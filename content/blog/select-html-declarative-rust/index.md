+++
title = "用声明式 Rust 选择 HTML 组件"
description = "有了 scraper_query，你不需要学习另一个语言（CSS）才能清理 HTML"
draft = false

weight = 9

[taxonomies]
tags = ["Rust"]

[extra]
feature_image = "declarative.png"
feature = true
+++

作为LLM 工程师和前端新手，我不是很懂 CSS，但我有时候确实要爬一些网页，解析、转换变成我的智能体的知识库。所以，我就想用，能不能用 Rust 来搞定这个？毕竟 Rust 是我最喜欢的语言。

我在网上简单查了一下，找到了 [`scraper`](https://github.com/rust-scraper/scraper) 这个库。它可以用来解析 HTML 文件，还可以用 CSS 选择器来选择 HTML 组件。看起来很简单，但是······什么是 CSS 选择器？我就只是想删一些组件，比如 `<meta>`、`<script>` 和带有 `promotion` 类的 `<div>`。

“冷”知识：

在 HTML 中，一个有两个类 `a` 和 `b` 的组件（比如 `<div>`）写作 `<div class="a b"></div>`。但是在 CSS 中怎么选择这个组件呢？要写 `div.a.b`！我这样习惯了面向对象语言，这种语法看起来有点奇怪。不过算了，规则就是规则，我可以自适应。

那在 CSS 里怎么选一个类名是 `a` 或 `b` 的 `<div>` 呢？它写作 `div.a, div.b`。GitHub Copilot 在补全上一句话的时候甚至给了我很多个别的选项！

我实际仔细看我的 HTML 文件的时候，就没这么简单了。我要选择并且删除这些组件：
1. 类名是 `promotion` 或 `ads` 的 `<div>`
2. 类名是 `search` 的 `<text>`
3. 类名不是 `link-to-documentation` 的 `<a>`
4. 类名是 `misc` 的 `<p>`，**除非**它同时也是 `documentation` 类
5. 所有是 `[li, ol, ul ...]` 其中之一的组件
6. ........ 这个列表有点长，我就不全列出来了

那我要怎么写 CSS 选择器来描述我想要的内容？而且我想用编程的方式动态更新这些筛选条件，那我就不得不操作字符串，但是这本质上是写得很混乱的元编程。
虽然但是，我确实**也这样做过了**！但是这种做法，在一个编程语言里，用字符串嵌入另一个 DSL（领域特定语言），写程序就会很痛苦。

> 我确实也知道很多库都是用 CSS 来选择 HTML 组件的。存在即合理。而且我知道像 [dom_query](https://docs.rs/dom_query/latest/dom_query/) 这样的库很有用。
> 不过我真的想先考虑易用性和简单性，而不是说性能或者单纯想学 CSS。

所以，我们先不管 CSS。已知的是：我们有一些 HTML 组件，它们有一些属性，比如名称、类名和 ID。我们想基于一些逻辑选择其中的一些。看起来我们只是需要一个查询引擎。或者更准确地说，一个 SAT 求解器（这是个冷笑话哈哈）

## 声明式语言的思考方式

我们很熟悉布尔值了。如果变量命名得当，逻辑表达式基本上就像读句子一样。比如说，`component.is_div() && component.is_class("a")` 看起来就很容易读出来。
但是这种面向对象风格的 API 跟 `scraper` 的 API 设计不太搭，因为用 `scraper` 的时候，我们用 `Node` 和 `NodeId` 来索引它的树结构，并且操作 HTML 文件和组件。

不过，我们可以让查询语句更加靠近声明式的风格。与其像 CSS 那样写 `div.a.b`，我们可以写 `Tag::Div & class("a") & class("b")`。这个逻辑语句几乎可以读出来了。

这就是 [`scraper_query`](https://crates.io/crates/scraper_query) 可以做到的。下面是一个完整的示例

```rust
use scraper::Html;
use scraper_query::*; // 使用 `HTMLIndex`、`Tag`、`class`、`id`
use markup5ever::interface::tree_builder::TreeSink;

let mut document = Html::parse_document(HTML);
let index = HTMLIndex::new(&document);
let query = Tag::Div & class("a") & class("b"); // 查找所有带有类 "a" 和 "b" 的 div
let node_ids = index.query(query);
// 简单的操作
for id in node_ids {
    document.remove_from_parent(&id);
}
```

现在我们可以使用 `&`、`|`、`!` 当然还有 `()` 来组合查询语句。如果还想要更明确的表达，我们可以写成 `Tag::Div.and(class("a").and(class("b")))` 这样。

> 有了 `&`、`|`、`!`，这个伪装在 Rust 逻辑表达式里的迷你 DSL 是图灵完备的！

## 实现原理

`scraper_query` 很简单，只有几百行代码。最重的部分，也就是查询引擎是 [Polars](https://pola.rs) 实现的。有了 Polars，只要：
1. 遍历所有 HTML 节点，建一个Polars DataFrame
    * 列分别是 `NODE_ID`、`CLASS`、`ID`（即 HTML 组件中的 `id`）和 `TAG`（即组件的名称）
    * 每个组件在表格中有一行记录
    * 这个表封装在 `HTMLIndex` 里面
2. 用像逻辑表达式这样的表达式查这个表，比如 `Tag::Div & class("a") & class("b")`
    * 这些表达式在 `std::ops::{BitAnd, BitOr, Not}` 的实现里被转换为 Polars 的表达式
    * 关于 Polars 表达式的详细信息，可以看 [`polars_plan::dsl`](https://docs.rs/polars-plan/latest/polars_plan/dsl/index.html)

例如：
```html
<div class="a b">
    <div class="c d">
        <p> Hello </p>
    </div>
    <div class="e" id="f">
        <p> World </p>
    </div>
</div>
```

这个 HTML 文件，在解析和构建完 `HTMLIndex` 之后，我们会有这样的 Polars 数据表：

| NODE_ID | TAG | CLASS | ID |
| :-----: | :-: | :---: | :-: |
| 0       | div | [ "a", "b" ] |    |
| 1       | div | [ "c", "d" ] |    |
| 2       | p   | []     |    |
| 3       | div | [ "e" ]    | "f"  |
| 4       | p   | []     |    |

如果我们用 `class("a") | class("c")` 这个语句来查询组件，本质上生成的 Polars 表达式是：

```rust
use polars::prelude::Expr;

let class_a_expr: Expr = col("CLASS").list().contains(lit("a"));
let class_c_expr: Expr = col("CLASS").list().contains(lit("c"));
let query_expr: Expr = class_a_expr.or(class_c_expr); // 这将用于过滤数据框
```

简单明了，但是能用。

## 之后的改进

一个是，Polars 这个依赖看起来可能太重了。毕竟它是一个的完整工具集，用来操作数据表。可能我可以只拿它的查询引擎来用。或者，专门为这个用例写一个小 SAT 求解器。

另一个是，我现在检查和清理 HTML 文件的过程不直观，也不能交互。可能我会写一个网页，基于像 `Tag::Div & class("a") & class("b")` 这样的语句来选择组件，然后即时渲染处理完的 HTML 文件。

## 元数据

版本：0.0.1

日期：2024.11.04

许可证：[CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/)