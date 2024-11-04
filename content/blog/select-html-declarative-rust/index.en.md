+++
title = "Select HTML Components in Declarative Rust"
description = "With scraper_query, you don't need to learn another language (CSS) to just clean up your HTML"
draft = false

weight = 9

[taxonomies]
tags = ["Rust"]

[extra]
feature_image = "declarative.png"
feature = true
+++

As a LLM engineer and a sloppy frontend newbie, I don't really know much about CSS, but I do need to crawl some web pages and turn them into a knowledgebase for my LLM agents. So, I thought, I'm going to do this in my favorite language, Rust :)

After some research, I found [`scraper`](https://github.com/rust-scraper/scraper), which parses HTML files. We can also select the HTML components with CSS selectors with it. Simple and easy, but wait, what's CSS selector? I just want to remove some components, like `<meta>`, `<script>` and `<div>` that is class `promotion`.

Here come some fun facts:
 
A component, say `<div>` that has two classes `a` and `b` in HTML is written as `<div class="a b"></div>`. But how can you select it with CSS? It's written as `div.a.b`! That seems a bit weird to a guy like me who got used to object-oriented dot notations. But OK, these are just conventions, I can work with them.

How about selecting a `<div>` that is either class `a` or `b` in CSS? It's written as `div.a, div.b`. My GitHub Copilot actually gave me multiple other options when completing the last sentence!

Things get a bit complicated when I actually take a closer look at my HTML files. I need to select and remove:
1. `<div>` that is either class `promotion` or `ads`
2. `<text>` that is class `search`
3. `<a>` that is not class `link-to-documentation`
4. `<p>` that is `misc` _unless_ it is also class `documentation`
5. all the components that are one of `[li, ol, ul ...]`
6. ........ this numbered list went a bit long, so I don't want to list it all.

How can I write CSS selectors to describe what I want? Also, I'd like to update these criteria programmatically, which means I have to do string manipulation that is essentially scrappy meta programming. 
I actually **did** it! But it's just painful when you embed a DSL inside a language in the form of strings.

> Yes, I know this is kind of the norm to use CSS to select HTML components, and I believe there are good reasons. I know crates like [dom_query](https://docs.rs/dom_query/latest/dom_query/) are super useful. 
> But I really want to focus on ergonomics and simplicity, not performance or curiosity about CSS.

So, let's forget CSS for now. We have some HTML components, which have some properties, like its name, class and id. We want to select some of them based on some logics. It sounds like we just need a query engine. Or, a SAT solver (no, it's a joke :)

## Let's think declaratively

I think we are pretty used to booleans. If naming is done right, logic expressions can be pretty declarative and literate. For instance, `component.is_div() && component.is_class("a")` seems quite readable. 
However, this object-oriented style of APIs doesn't fit with the design of `scraper`. Working with `scraper`, we manipulate the HTML file and its components with `Node` and `NodeId`, which help us index into the tree structure.

Despite that, we can make a query more declarative. Instead of `div.a.b`, we can write `Tag::Div & class("a") & class("b")`. You can almost read the logic as a sentence.

This is what [`scraper_query`](https://crates.io/crates/scraper_query) allows. The complete sample is

```rust
use scraper::Html;
use scraper_query::*; // use `HTMLIndex`, `Tag`, `class`, `id`
use markup5ever::interface::tree_builder::TreeSink;

let mut document = Html::parse_document(HTML);
let index = HTMLIndex::new(&document);
let query = Tag::Div & class("a") & class("b"); // find all divs with class "a" and "b"
let node_ids = index.query(query);
// simple manipulation
for id in node_ids {
    document.remove_from_parent(&id);
}
```

Now you can use `&`, `|`, `!` and of course `()` to compose your query. If you prefer to be even more explicit, you can do something like `Tag::Div.and(class("a").and(class("b")))`.

> With `&`, `|`, `!`, this little DSL disguised in Rust logic expressions is Turing-complete!


## Under the hood

`scraper_query` is pretty simple. It has only a few hundreds of lines of code. The heavy lifting query engine is implemented in [Polars](https://pola.rs). With Polars, all I need is to:
1. Iterate through all HTML nodes and build a Polars DataFrame which is conceptually a table
    * The columns are `NODE_ID`, `CLASS`, `ID` (i.e., `id` in HTML components) and `TAG` (i.e., the name of a component)
    * Each component has one row in the table
    * The table is encapsulated in `HTMLIndex`
2. Query the data frame with expressions like logic expressions, like `Tag::Div & class("a") & class("b")`
    * These are transformed into Polars expressions in `std::ops::{BitAnd, BitOr, Not}` implementations.
    * For details of Polars expressions, see [`polars_plan::dsl`](https://docs.rs/polars-plan/latest/polars_plan/dsl/index.html).

Simple and clean, it serves its purpose.

## Future Work

For one, the dependency, Polars, might seem too heavy. It's full fleet of tools built for dataframes after all. Probably we can just strip out the query engine of it. Or, build a little SAT solver just for this use case.

For another, I find the process of examining and cleaning HTML files is not straightforward and interactive. Probably I will implement a web app that interactively and responsively shows the rendering of a processed HTML file based on selection rules like `Tag::Div & class("a") & class("b")`.


## Metadata

Version: 0.0.1

Date: 2024.11.04

License: [CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/)
