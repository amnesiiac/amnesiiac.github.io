---
layout: post
title: "daily writing tips (blog)"
author: "twistfatezz"
date: 2023-04-13 20:42
categories: "2023"
tags:
  - blog
  - ongoing
---

### # raw tag to disable liquid processing
following shows a snippet which has conflicts in liquid:
{% raw %}
```cpp
vector<int> vec = {{1,3}, {1,2}, {2,4}};
```
```txt
Error: Liquid syntax error (line 6): Variable '{{1,3}' was not properly terminated with regexp: /\}\}/
Error: Run jekyll build --trace for more information.
```
{% endraw %}
jekyll blog first using liquid to resolve the markdown, then the markdown interpretor. when there's conflicts in blog with liquid, use the "raw" tag defined in [liquid doc](https://shopify.github.io/liquid/tags/template/#raw), can bypass the liquid resolving process.

<hr>

### # embed pics inside markdown
<details>
    <summary><b>state transition model</b></summary>
    <center><img src="/assets/images/2022/markov-chain-monte-carlo/1.png" width="50%"></center>
</details>

<hr>

### # usefull elements in tui picture 
```txt
todo
```

<hr>

### # how to embed svg inside jekyll blog
just use the svg tag:
```text
<svg width="400" height=300>
    <circle cx="150" cy="100" r="10" fill="blue"/>
</svg>
```
the following are not necessary:
```text
{::nomarkdown}
<svg width="400" height=300>
    <circle cx="150" cy="100" r="10" fill="blue"/>
</svg>
{:/}
```
