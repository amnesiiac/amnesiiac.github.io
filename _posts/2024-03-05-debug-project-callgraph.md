---
layout: post
title: "project function call graph (code, reverse engineering)"
author: "melon"
date: 2024-03-05 21:02
categories: "2024"
tags:
  - code insight
---

call graph shows the graphical relationship between the function calls within a program,
allowing developers to understand the program flow and identify performance issues.

static analysis and dynamic analysis are two methods for analysis the project callgraph.
static way analysis the code of a program without execution,
dynamic way analysis the project as it executes.

<hr>

### # dynamic call graph generators
1 pycallgraph3: a reverse engineering tool for python
```text
$ pip install pycallgraph3
```

a simple example to illustrate the usecase format:
```text
from pycallgraph import PyCallGraph
from pycallgraph.output import GraphvizOutput

graph = GraphvizOutput()
graph.output_file = "file4.png"

with PyCallGraph(output=graph):
    print("Hello world")
```
todo: test for hostfw code call graph

<p style="margin-bottom: 20px;"></p>

2 valgrind (to be confirmed)
```text
$ valgrind --tool=callgrind --log-file=callgrind.log ${binary_file}  # generate call graph data
$ kcachegrind callgrind.out.x                                        # show the graph out
```

<p style="margin-bottom: 20px;"></p>

3 radare2: a tui based reverse engineering & cmdline toolset (todo)  
ref: https://github.com/radareorg/radare2

<hr>

### # static call graph generators
1 pycg + pyvis: a tool for python static call graph
```text
$ pip install pycg    # collect call graph data
$ pip install pyvis   # plot the call graph
```

<p style="margin-bottom: 20px;"></p>

2 cally: a tool for c function static call graph generator  
ref: https://github.com/chaudron/cally
```text
1 use -fdump-rtl-expand in makefile cflags to dump gcc rtl files
2 use -fno-inline-functions -O0 to include static function into callgraph
3 use gcc generated rtl file (.expand) and pass them to cally.py
4 todo: how to make expand file at certian folder
```

1) add -fdump-rtl-expand to udrv/makefile/cflags:
```text
--- a/Makefile  Tue Apr 23 16:31:43 2024 +0800
+++ b/Makefile  Thu May 16 17:10:51 2024 +0800
@@ -32,7 +32,7 @@
         -funwind-tables          \
         -fsigned-char

-CFLAGS  += -g -Wall -O0 -Werror
+CFLAGS  += -g -Wall -O0 -Werror -fdump-rtl-expand
 ifeq ($(UDRV_FEATURE_PSEUDO_SPI_FULL_DUPLEX),y)
 override CFLAGS += -DFEATURE_PSEUDO_SPI_FULL_DUPLEX
 endif
```

2) follow udrv documentation/simulation, run make to compile the whole project
maintain all the generated rtl expand files at udrv/rlt folder.

3) generate callgraph for single gcc rtl expand file:
```text
$ python3 cally.py ./rtl/udrv_rip.c.166r.expand | dot -Grankdir=LR -Tpng -o single_expand_callgraph.png
```

<img src="https://cdn.jsdelivr.net/gh/slothfull/cdn@main/image/cg2.pdf" width="360"/>

4) generate full callgraph with designated caller as startup node  
e.g. get all the graph originates from udrv_read_write_rip:
```text
$ find ./rtl/ -name *.expand | xargs ./cally.py --caller udrv_read_write_rip | dot -Grankdir=LR -Tpng -o callergraph.png
```

<img src="https://cdn.jsdelivr.net/gh/slothfull/cdn@main/image/cg.pdf" width="750"/>

5) generate full callgraph with designated callee as endup node
```text
$ find ./rtl/ -name *.expand | xargs ./cally.py --callee udrv_read_write_rip | dot -Grankdir=LR -Tpng -o calleegraph.png
```

<img src="https://cdn.jsdelivr.net/gh/slothfull/cdn@main/image/cg3.pdf" width="400"/>
