---
layout: post
title: "daily writing tips (blog)"
author: "melon"
date: 2023-04-13 20:42
categories: "2023"
tags:
  - blog
---

### # disable liquid processing by raw tag
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

### # code blocks
$ add self-custom code block style (e.g. blur text):

```text
<pre><code style="filter: blur(1px);">
#include <stdio.h>

int main(){
    printf("hello world!");
    return 0;
}
</code></pre>
```

$ customize normal text in blog:

```text
<style>
k{
    filter: blur(2px);
}
</style>

<k>you guess what's here</k>
```

<hr>

### # insert graph in blog post
$ insert graph with toggle button:

```text
<details>
    <summary><b>state transition model</b></summary>
    <center><img src="/assets/images/2022/markov-chain-monte-carlo/1.png" width="50%"></center>
</details>
```

$ insert math graph hosted on https://www.desmos.com/calculator:

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

$ using github cdn repo as graph registry, with width set:

```text
<img src="https://cdn.jsdelivr.net/gh/slothfull/cdn@main/image/autoencoder.pdf" width="350"/>
```

<hr>

### # usefull elements in tui picture
search for tui in blog post for guidance.

<hr>

### # embed svg in blog post
```text
<svg width="400" height=300>
    <circle cx="150" cy="100" r="10" fill="blue"/>
</svg>
```

<hr>

### # control the division null line height
1 use special escape character

```text
the first blog line.
&nbsp;
the second blog line.
```

2 use null text block with margin customized:

```text
<p style="margin-bottom: 20px;"></p>
```

<hr>

### # blog link remote url format
1 markdown remote link anchor implicitly  
this is a url which allow actual hidden url at bottom of blog: [url anchor][id].
[id]: http://groups.google.com/group/celluloid-ruby

```text
remote link anchor to enable set long url at bottom of blog [url anchor][id].
[id]: http://groups.google.com/group/celluloid-ruby
```
