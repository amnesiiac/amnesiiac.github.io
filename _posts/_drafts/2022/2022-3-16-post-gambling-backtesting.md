---
layout: post
title: "backtesting for stocks gambling (回测)"
subtitle: '[backtesting] - 策略自动回测工具' 
author: "twistfatezz"
header-style: text
# header-img: "img/in-post/economics_1/headimg.png"
# header-mask: 0.8
comments: false 
mathjax: false 
date: 2022-03-16 11:30
lang: ch 
catalog: true 
categories: gambling 
tags:
  - Time 2022
---
### Backtest Link
Backtest for simple SMAcross strategy [click to see details](/html/test.html).

### Backtest Iframe
Mention: the iframe below only works on localhost://...
<html lang="en">
<head>
    <div class="snippet" style="background: white">
        <iframe data-src="http://localhost:4000/html/test.html" id="iframe" loading="lazy" 
            style="width:100%; margin-top:1em; height:800px; overflow:hidden; transform-origin:left top; transform:scale(1,1)" 
            data-ga-on="wheel" 
            data-ga-event-category="iframe" data-ga-event-action="wheel"></iframe>
        <script>
            var iframe = document.getElementById('iframe');
            if(!window.IntersectionObserver){
                window.addEventListener('load', () => {
                    iframe.src = iframe.dataset.src;
                });
            }
            else{
                (new IntersectionObserver((entries, observer) => {
                    if(entries[0].intersectionRatio === 0)
                        return;
                    iframe.src = iframe.dataset.src;
                    observer.disconnect();
                },{
                    threshold: .01
                })).observe(iframe);
            }
        </script>
    </div>
</head>
</html>

## Reference
> https://kernc.github.io/backtesting.py/ <br>
> https://github.com/kernc/backtesting.py

> 1 当使用inline数学公式且公式经过GFM排版之后都在同一行 使用`$...$`符号<br>
> 2 当希望数学公式单独成行或者经过GFM排版之后占用多行 应当使用`$$...$$`符号<br>
> 3 对于表示条件概率 需要表示竖线的时候`|` 应当使用`\mid` 而不是直接在键盘上打出`|` => 容易被编辑器认为是一个md制表符<br>
> 4 在md引入图片的时候 不要使用`<center>`和`</center>` 在这篇文档的编辑过程中vscode的preview插件在使用了上述符号之后 导致下一段的数学公式预览显示不正常<br>
> 5 使用md的时候 单独的两段文字上下需要空出一行<br>
> 6 想要强制换行的时候 需要使用`<br>`而不是`<enter>`<br>
> 7 特殊字符如果想要避免和md解析关键字冲突 应当使用\`\`将关键字包含在内 <br>
> 8 `<center><img src="/img/in-post/economics_4/xxx.png" width="60%"></center>` <br>
> 9 使用html设置图片文字环绕方式: <br>
    `<div>` <br>
        `<img src="/img_path" align="left" width="40%" hspace="" vspace=""/>` <br>
        `<p>paragraph1 around the picture</p>` <br>
        `<p>paragraph2 around the picture</p>` <br>
        `<p>paragraph3 around the picture</p>` <br>
    `</div>` <br>
> 10 `<font style="color:red; font-weight:bold">加粗蓝色</font>`用来设置字体颜色 <br>
> 11 使用html设置可折叠部分内容：<br>
  `<details>` <br>
      `<summary><b>[点击展开] xxx</b></summary>` <br>
      `<center><img src="/img/in-post/tcp-ip_img/tcp_20_9.pdf" width="100%"></center>` <br>
  `</details>` <br>
> 12 问题脚注: ### ???problem😫problem???  大标题标识符：▶︎ 小标题标识符：➤ <br>
> 13 实现文章排版tab：`&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;` <br>
> 14 项目序号 ⓪➀➁➂➃➄➅➆➇➈➉ ➊➋➌➍➎➏➐➑➒➓
