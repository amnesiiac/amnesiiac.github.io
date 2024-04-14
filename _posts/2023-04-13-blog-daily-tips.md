---
layout: post
title: "daily writing tips (blog)"
author: "melon"
date: 2023-04-13 20:42
categories: "2023"
tags:
  - blog
---

### # raw tag to disable liquid processing
consider the following blog code snippet:
{% raw %}
```cpp
vector<int> vec = {{1,3}, {1,2}, {2,4}};
```

which will result in:
```text
Error: Liquid syntax error (line 6): Variable '{{1,3}' was not properly terminated with regexp: /\}\}/
Error: Run jekyll build --trace for more information.
```
{% endraw %}
jekyll blog first using liquid to resolve the markdown, then the markdown interpretor itself.

when there's conflicts in blog with liquid, use the "raw" tag defined in
[liquid doc](https://shopify.github.io/liquid/tags/template/#raw),
can bypass the liquid resolving process.

<hr>

### # insert graph in markdown
insert graph with toggle button:
```text
<details>
    <summary><b>state transition model</b></summary>
    <center><img src="/assets/images/2022/markov-chain-monte-carlo/1.png" width="50%"></center>
</details>
```

insert math graph hosted on https://www.desmos.com/calculator:
```text
<details>
    <summary><b>Solve the Integral From a to b</b></summary>
    <center>
    <iframe src="https://www.desmos.com/calculator/vvpzvriqdq?embed" width="400" height="180"
            style="border: 1px solid #ccc" frameborder=0>
    </iframe>
    </center>
</details>
```

using github cdn repo as graph registry, with width set:
```text
<img src="https://cdn.jsdelivr.net/gh/slothfull/cdn@main/image/autoencoder.pdf" width="350"/>
```

<hr>

### # usefull elements in tui picture 
see gui element for guidance.

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

<hr>

### # insert a null line between the first & second blog line
the first blog line.
<br/>

the second blog line.

which is implementated as:
```text
the first blog line.
<br/>

the second blog line.
```
