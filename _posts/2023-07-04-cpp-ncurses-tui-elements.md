---
layout: post
title: "tui elements for graphic simulation (c++ ncurses)"
author: "melon"
date: 2023-07-04 21:36
categories: "2023"
tags:
  - cpp
  - ncurses
---

### # TUI program ouput
```txt
Upper left corner ┌
Upper right corner ┐
Lower left corner └
Lower right corner ┘
Tee pointing right ├
Tee pointing left ┤
Tee pointing up ┴
Tee pointing down ┬
Horizontal line ─
Vertical line │
Large Plus or cross over ┼
Scan Line 1 ⎺
Scan Line 3 ⎻
Scan Line 7 ⎼
Scan Line 9 ⎽
Diamond ◆
Checker board (stipple) ▒
Degree Symbol °
Plus/Minus Symbol ±
Bullet ·
Arrow Pointing Left <
Arrow Pointing Right >
Arrow Pointing Down v
Arrow Pointing Up ^
Board of squares #
Lantern Symbol ␋
Solid Square Block #
Less/Equal sign ≤
Greater/Equal sign ≥
Pi π
Not equal ≠
UK pound sign £
```

<hr>

### # code
{% raw %}
```cpp
#include <ncurses.h>

int main(){
    initscr();
    printw("Upper left corner "); addch(ACS_ULCORNER); printw("\n");
    printw("Upper right corner "); addch(ACS_URCORNER); printw("\n");
    printw("Lower left corner "); addch(ACS_LLCORNER); printw("\n");
    printw("Lower right corner "); addch(ACS_LRCORNER); printw("\n");
    printw("Tee pointing right "); addch(ACS_LTEE); printw("\n");
    printw("Tee pointing left "); addch(ACS_RTEE); printw("\n");
    printw("Tee pointing up "); addch(ACS_BTEE); printw("\n");
    printw("Tee pointing down "); addch(ACS_TTEE); printw("\n");
    printw("Horizontal line "); addch(ACS_HLINE); printw("\n");
    printw("Vertical line "); addch(ACS_VLINE); printw("\n");
    printw("Large Plus or cross over "); addch(ACS_PLUS); printw("\n");
    printw("Scan Line 1 "); addch(ACS_S1); printw("\n");
    printw("Scan Line 3 "); addch(ACS_S3); printw("\n");
    printw("Scan Line 7 "); addch(ACS_S7); printw("\n");
    printw("Scan Line 9 "); addch(ACS_S9); printw("\n");
    printw("Diamond "); addch(ACS_DIAMOND); printw("\n");
    printw("Checker board (stipple) "); addch(ACS_CKBOARD); printw("\n");
    printw("Degree Symbol "); addch(ACS_DEGREE); printw("\n");
    printw("Plus/Minus Symbol "); addch(ACS_PLMINUS); printw("\n");
    printw("Bullet "); addch(ACS_BULLET); printw("\n");
    printw("Arrow Pointing Left "); addch(ACS_LARROW); printw("\n");
    printw("Arrow Pointing Right "); addch(ACS_RARROW); printw("\n");
    printw("Arrow Pointing Down "); addch(ACS_DARROW); printw("\n");
    printw("Arrow Pointing Up "); addch(ACS_UARROW); printw("\n");
    printw("Board of squares "); addch(ACS_BOARD); printw("\n");
    printw("Lantern Symbol "); addch(ACS_LANTERN); printw("\n");
    printw("Solid Square Block "); addch(ACS_BLOCK); printw("\n");
    printw("Less/Equal sign "); addch(ACS_LEQUAL); printw("\n");
    printw("Greater/Equal sign "); addch(ACS_GEQUAL); printw("\n");
    printw("Pi "); addch(ACS_PI); printw("\n");
    printw("Not equal "); addch(ACS_NEQUAL); printw("\n");
    printw("UK pound sign "); addch(ACS_STERLING); printw("\n");
    refresh();
    getch();
    endwin();
    return 0;
}
```
{% endraw %}

<hr>

### # Makefile to build this app
remember to use -lncurses/-lform to enable link to the ncurse lib.
```makefile
CXX=g++
CFLAGS=-g -Wall -std=c++11 -pthread -w
OUT=out

SUBDIR=$(shell ls -d */)

ROOTSRC=$(wildcard *.cpp)
ROOTOBJ=$(ROOTSRC:%.cpp=%.o)

SUBSRC=$(shell find $(SUBDIR) -name '*.cpp')
SUBOBJ=$(SUBSRC:%.cpp=%.o)

$(OUT):$(ROOTOBJ) $(SUBOBJ)
	$(CXX) $(CFLAGS) -o $@ $^ -lncurses
.c.o:
	$(CXX) $(CFLAGS) -c $< -o $@

.PHONY:clean all
clean:
	rm -f *.o $(OUT) $(ROOTOBJ) $(SUBOBJ)
```

<hr>
