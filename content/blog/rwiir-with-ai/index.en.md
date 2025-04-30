+++
title = "Rewrite It in Rust with AI - Oxidize a Portfolio Website in a Day"
description = "With Devin and Cursor, rewrite 4K lines of React code to Dioxus in one day"
draft = false

weight = 10

[taxonomies]
tags = ["Rust"]

[extra]
feature_image = "rusty_web.png"
feature = true
+++

Recently, I have a larger project in mind and I think I really need an AI agent to help me. Despite I'm an engineer working on AI, I've never pushed it to its limit.
For that large project, I really need to know where AI can fail without noticing it's failing itself so that I can get its back. So, I decided to test it out in a smaller project.

Luckily, there's a side project that I have been procrastinating to do. My portfolio website was based on the code of [MasterPortfolio](https://github.com/ashutosh1919/masterPortfolio), which was written in JS with React.
As a fan of Rust, I wanted to rewrite it in Rust, but it has 4K lines of JS and 2K lines of CSS. Despite the size, the codebase is well-structured and clean. So, I decided to let an agent do that and I picked [Devin](https://devin.ai). This is the start of my story.

Before getting into the story, here is a quick announcement: The oxidized codebase of my portfolio website is opensourced as [hack-portfolio](https://github.com/ifsheldon/hack-portfolio). It is powered by [Dioxus](https://dioxuslabs.com).
And I _manually_ refactored the code so that it is hackable. By "hackable", I mean the codebase is structured and uses no black magics. You can easily customize it with your own content, style and UI components.
It's a great scaffold and starting point for Rustaceans ❤ to develop their own portfolio websites. Please feel free to file issues and/or make PRs!

Back to the story. The old codebase uses plain CSS and Javascript with React. These technologies are well learned by any good LLMs. However, Dioxus, the Rust frontend framework I chose, is rather young, and it is making a lot of breaking changes in their rapid development.
Even if the LLMs behind Devin know about Dioxus, the knowledge is likely outdated. So, one preparation I made is to pull the source code of Dioxus documentation and make it a submodule.

The file structure is like below:

```shell
portfolio_js
├── ..... old js codebase
├── dioxus_docsite
    └── .... source code of Dioxus documentation
└── portfolio_rs
    └── dev.txt
    └── .... source code TODO
```

`dev.txt` is for Devin as well. I used Claude 3.7 Sonnet Thinking in Cursor to write this document. I told it to browse through the entire JS codebase **and** Dioxus documentation to write a concise development plan for this complete rewrite. For those interested, you can take a look at the
content [here](dev.txt).

> For those who are interested in Devin but hesitate to try it out due to the $500/month subscription price, Devin now has the pay-as-you-go option!

After finishing these preparations, I signed up Devin, topped up $20.25 for 9 ACUs and linked it to my repo. The prompt I gave is simple as hell, just like a boss would said :)

```
Help me rewrite this codebase of my portfolio website with Rust and Dioxus. 
Detailed instructions can be found in `dev.md`
```

When it started working, basically Devin got its own computer with VSCode and Chrome. You can interact with that computer if you like. For example, installing a tool for Devin or stopping it from doing stupid things.
And of course, you can watch along. It felt kind of unreal when you chose to "Follow Devin", which shows what Devin is working on, typing or looking at. The code unfolded into multiple files quickly. Now you can grab a cup of coffee and enjoy your time.
But I watched it working for an hour :)

When Devin is busy, you can still give it extra instructions and/or tips, and it will take notes in its "Suggested Knowledge". Knowing Devin is taking note is kind of nice experience.

After one hour, Devin seemed to have finished the code and made a PR, so I took over its computer and checked the result on Chrome. It was simply crap :(

The general layout was almost done, but anything else was not good. Since Devin can actually see the graphical outcome on Chrome, I ran `npm run start` to render my old JS website and `dx serve` to render the Dioxus version, and I told it to compare the results and fix things.

That worked! I watched it switching between the new and old websites, comparing the differences and fixing code for another half an hour. Until it got stuck (of course). It got stuck when trying to fix image problems. The images on the website didn't show up and it tried different methods.
However, there were other issues that are more critical, like layouts were not quite right. So, I simply told it that forget images for now, fix critical issues like layouts first. It fixed those critical issues in ~5mins :)

When critical ones got fixed, it got stuck in the minor issues (like images) again, trying different solutions. That's when I stopped Devin and took over the work.

After that, I spent 6 hours with Cursor to fix and polish the code, which is another story. But long story short, Devin got stuck in image issues because it tried to solve the problem in a React way, while the asset system in Dioxus is totally different.
It should have taken a step back, looked up the documentation and then come up a Dioxus solution. In other words, it didn't know (or realize) it didn't know (Dioxus asset system). And that's when AI agents fail without realizing it's failing.

As an insider, I kind of expect such failure beforehand, but does this mean AI cannot do anything? No.

Let's take a step back. Although I could not get the perfect result in one go, it helped me finish >60% of the work so that I didn't have to write code from scratch.
Moreover, I spent ~9 hours in total for this rewrite, which would have costed me probably a week. It's 4K lines of JS + 2K lines of CSS, after all.

Besides such an experience, I know more about the subtle limitations of current AI agents and here are my advice:

* For a big project for AI, you can write a detail document about the plan. You don't need to write it yourself and AI can help.
* Before starting the work, try to think about what AI does not know or the knowledge that may be outdated. Explicitly ask AI to refer to documentation.
* Always explicitly ask AI to validate their outcome and find issues early. For example, "Run cargo check every time you finish one piece of work" or "Compare the UIs and fix issues"
* For now, AI is not going to give you perfect results end-to-end. You should expect that sometimes they get stuck, and you should fix some issues yourself.
