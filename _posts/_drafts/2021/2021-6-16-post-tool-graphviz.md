---
layout: post
title: "Tool - graphviz"
subtitle: '流程图绘制工具graphviz相关知识整理'
author: "twistfatezz"
header-style: text
# header-img: "img/in-post/economics_1/headimg.png"
# header-mask: 0.8
comments: false 
mathjax: false 
date: 2021-06-16 16:08
lang: ch 
catalog: true 
categories: tool 
tags:
  - Time 2021
---
## Sample-1
<center><img src="/img/in-post/tool_img/graphviz_1.pdf" width="40%"></center>

```dot
digraph G{
    size ="4,4";
    main [shape=box];   /* this is a comment */
    main -> parse [weight=8];
    parse -> execute;
    main -> init [style=dotted];
    main -> cleanup;
    execute -> { make_string; printf}
    init -> make_string;
    edge [color=red];   // so is this
    main -> printf [style=bold,label="100 times"];
    make_string [label="make a\nstring"];
    node [shape=box,style=filled,color=".7 .3 1.0"];
    execute -> compare;
}
```

## Sample-2
<center><img src="/img/in-post/tool_img/graphviz_2.pdf" width="30%"></center>

```dot
digraph G{
    fontname="Bitstream Vera Sans"
    fontsize=8
    node[
        fontname="Bitstream Vera Sans"
        fontsize=8
        shape="record"
    ]
    edge[
        fontname="Bitstream Vera Sans"
        fontsize=8
    ]
    Animal[
        label="{Animal|+ name : string\l+ age : int\l|+ die() : void\l}"
    ]
    subgraph clusterAnimalImpl{
        label="Package animal.impl"
        Dog[
            label="{Dog||+ bark() : void\l}"
        ]

        Cat[
            label="{Cat||+ meow() : void\l}"
        ]
    }
    edge[
        arrowhead="empty"
    ]
    Dog -> Animal
    Cat -> Animal
    edge[
        arrowhead="none"
        headlabel="0..*"
        taillabel="0..*"
    ]
    Dog -> Cat
}
```

### Sample-3
<center><img src="/img/in-post/tool_img/graphviz_3.pdf" width="80%"></center>

```dot
digraph G{
    fontname = "Bitstream Vera Sans"
    fontsize = 8
    node[
        fontname="Bitstream Vera Sans"
        fontsize=8
        shape="record"
    ]
    edge[
        fontname="Bitstream Vera Sans"
        fontsize=8
    ]
    zooAnimal[
        label="{zooAnimal | 
            × bool _onExhibit \l × string _name \l × string _family_name \l |
            + zooAnimal(string name, bool onExhibit, string family_name) \l
            + virtual ~zooAnimal() \l
            + virtual ostream& print(ostream&) const \l
            + bool onExhibit(...) \l
            + string name() const \l
            + string family_name() const \l
            }"
    ]
    Bear[
        label="{Bear |
            × DanceType _dance \l
            + enum DanceType\{two_left_feet, macarena, fandango, waltz\} \l |
            × Bear():dance(two_left_feet) \l
            + Bear(string name, bool onExhibit=true) \l
            + bool onExhibit(...) \l
            + virtual ostream& print(ostream&) const \l
            + void dance(DanceType) \l
            }"
    ]
    Panda[
        label="{Panda |
            × bool _sleeping \l | 
            + Panda(string name, bool onExhibit=true):... \l
            + virtual ostream& pring(ostream&) const \l
            + bool sleeping() const \l
            + void sleeping(bool newval) \l
        }"
    ]
    toyAnimal[
        label="{toyAnimal}"
    ]
    BookCharacter[
        label="{BookCharacter}"
    ]
    Character[
        label="{Character}"
    ]
    edge[
        arrowhead="empty"
    ]
    Panda -> Bear
    Panda -> BookCharacter
    BookCharacter -> Character
    edge[
        arrowhead="empty"
        style="dashed"
    ]
    Bear -> zooAnimal 
    Panda -> toyAnimal
}
```

## Reference
> https://gitlab.com/graphviz/graphviz/-/blob/main/doc/dotguide.pdf <br>
> http://www.ffnn.nl/pages/articles/media/uml-diagrams-using-graphviz-dot.php <br>
> http://www.graphviz.org <br> 

> 1 当使用inline数学公式且公式经过GFM排版之后都在同一行 使用`$...$`符号<br>
> 2 当希望数学公式单独成行或者经过GFM排版之后占用多行 应当使用`$$...$$`符号<br>
> 3 对于表示条件概率 需要表示竖线的时候`|` 应当使用`\mid` 而不是直接在键盘上打出`|` => 容易被编辑器认为是一个md制表符<br>
> 4 在md引入图片的时候 不要使用`<center>`和`</center>` 在这篇文档的编辑过程中vscode的preview插件在使用了上述符号之后 导致下一段的数学公式预览显示不正常<br>
> 5 使用md的时候 单独的两段文字上下需要空出一行<br>
> 6 想要强制换行的时候 需要使用`<br>`而不是`<enter>`<br>
> 7 特殊字符如果想要避免和md解析关键字冲突 应当使用``将关键字包含在内 <br>
> 8 `<center><img src="/img/in-post/economics_4/xxx.png" width="60%"></center>`
> 9 使用html设置图片文字环绕方式: <br>
    `<div>` <br>
        `<img src="/img_path" align="left" width="40%" hspace="" vspace=""/>` <br>
        `<p>paragraph1 around the picture</p>` <br>
        `<p>paragraph2 around the picture</p>` <br>
        `<p>paragraph3 around the picture</p>` <br>
    `</div>` <br>
