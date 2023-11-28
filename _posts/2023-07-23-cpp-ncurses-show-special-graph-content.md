---
layout: post
title: "show special char(tui graph) in ncurses (c++ ncurses)"
author: "melon"
date: 2023-07-23 20:03
categories: "2023"
tags:
  - cpp
  - ncurses
---

{% raw %}
### # program read local file(contains special chars) and output to ncurses screen
```cpp
#include <ncurses.h>
#include <string>
#include <unistd.h> // sleep
#include <locale.h>
#include <wchar.h>  // wchar type judge (alphabet, space ...)

using std::string;

void show(WINDOW* win, string file){
    FILE* fp;
    wchar_t c;
    fp = fopen(file.c_str(), "r") ; // opening an existing file
    if(fp == NULL){ // show nothing
        return;
    }
    int row = 0; int col = 0;  // char pos
    while(1){
        c = getwc(fp); // get wchar from file stream
        if(c == EOF){ // end of file
            break;
        }
        // deal with special tui-elements wchar_t
        if(c == L'─'){
            mvwaddch(win, row, col, ACS_HLINE);
            col++;
        }
        else if(c == L'┌'){
            mvwaddch(win, row, col, ACS_ULCORNER);
            col++;
        }
        else if(c == L'└'){
            mvwaddch(win, row, col, ACS_LLCORNER);
            col++;
        }
        else if(c == L'┘'){
            mvwaddch(win, row, col, ACS_LRCORNER);
            col++;
        }
        else if(c == L'┐'){
            mvwaddch(win, row, col, ACS_URCORNER);
            col++;
        }
        else if(c == L'│'){
            mvwaddch(win, row, col, ACS_VLINE);
            col++;
        }
        else if(c == L'├'){
            mvwaddch(win, row, col, ACS_LTEE);
            col++;
        }
        else if(c == L'├'){
            mvwaddch(win, row, col, ACS_LTEE);
            col++;
        }
        else if(c == L'┤'){
            mvwaddch(win, row, col, ACS_RTEE);
            col++;
        }
        else if(c == L'┴'){
            mvwaddch(win, row, col, ACS_BTEE);
            col++;
        }
        else if(c == L'┬'){
            mvwaddch(win, row, col, ACS_TTEE);
            col++;
        }
        else if(c == L'┼'){
            mvwaddch(win, row, col, ACS_PLUS);
            col++;
        }
        else if(c == '\n'){ // newline
            row++; col=0;   // reset
            mvwprintw(win, row, col, "%c", c);
        }
        else{ // any chars
            mvwprintw(win, row, col, "%c", c); // 0,0 -> 0,1
            col++; // move
        }
    }
    wrefresh(win);
    fclose(fp);  // close 
}

int main(int argc, char **argv){
    setlocale(LC_ALL, ""); // clib use local from system default
    // init
    initscr();
    cbreak();
    noecho();

    WINDOW* my_win=newwin(120, 120, 0, 0);  // win

    show(my_win, "test"); // show

    sleep(10);  // holdon

    endwin();
    return 0;
}
```

<hr>

### # the program utilized file: text
```txt
┌────────────────────────┐            ┌───────────┐         ┌──────────────┐
│ user process(blocking) │<----(10)---┤ ksoftirqd │<--(7)---┤ software irq │<---------┐
└─────────────┬───┬──────┘            └─────┬─────┘         └──────────────┘          |
              |   |                         |                                         |
              |   | 2. wait for recv        |                                         |
1. create sock|   └-----------------┐       |8. get packets from                      |
    ┌─────────┼─────────────────────┼───────┼─────┐ ringbuffer                        |
    │┌────────+────────┐       waiting que  |     │                                   |
    ││socket kernel obj├──┐  ┌─┬─┬─┬+┬───┬─┐|     │                                   |
    │└────────┬────────┘  └─>│ │ │ │ │   │ │|     │                                   |
    │         |              └─┴─┴─┴─┴───┴─┘|     │                6. handling hw irq |
    │socket recv que                        |     │                                   |(6)
    │┌─┬─┬─┬─┬+──┬─┐     ┌───┐        ┌─────+────┐│ ┌─────┐  ┌──────────────┐      ┌──┴──┐
    ││ │ │ │ │...│ │<----┤skb│<-------┤ringbuffer├┼─┤  ?  ├──┤ bridge/memory├──────┤ cpu │
    │└─┴─┴─┴─┴───┴─┘     └───┘        └─────+────┘│ └─────┘  │  controller  │      └──+──┘
    │9. push into socket recv que        (4)|     │          └──────────────┘  ┌──────┴───────┐
    └───────────────────────────────────────┼─────┘                            │ hardware irq │
                   4. DMA frames to mem.    |   [PCIe]                         └──────+───────┘
────────────────────────────────────────────|─────────────────────────────────────────|──────────
               3. packets send to nic    ┌──┴──┐  5. issue hardware irq notify cpu    |
               ------------------------->│ nic ├--------------------------------------┘
                                         └─────┘
switch the focus window when using gdb -tui
-------------------------------------------
# method-1
press keys 'ctrl+x; o'   # cycle through windows

# method-2
$ focus next
```

<hr>

### # the program output:
```txt
┌────────────────────────┐            ┌───────────┐         ┌──────────────┐
│ user process(blocking) │<----(10)---┤ ksoftirqd │<--(7)---┤ software irq │<---------┐
└─────────────┬───┬──────┘            └─────┬─────┘         └──────────────┘          |
              |   |                         |                                         |
              |   | 2. wait for recv        |                                         |
1. create sock|   └-----------------┐       |8. get packets from                      |
    ┌─────────┼─────────────────────┼───────┼─────┐ ringbuffer                        |
    │┌────────+────────┐       waiting que  |     │                                   |
    ││socket kernel obj├──┐  ┌─┬─┬─┬+┬───┬─┐|     │                                   |
    │└────────┬────────┘  └─>│ │ │ │ │   │ │|     │                                   |
    │         |              └─┴─┴─┴─┴───┴─┘|     │                6. handling hw irq |
    │socket recv que                        |     │                                   |(6)
    │┌─┬─┬─┬─┬+──┬─┐     ┌───┐        ┌─────+────┐│ ┌─────┐  ┌──────────────┐      ┌──┴──┐
    ││ │ │ │ │...│ │<----┤skb│<-------┤ringbuffer├┼─┤  ?  ├──┤ bridge/memory├──────┤ cpu │
    │└─┴─┴─┴─┴───┴─┘     └───┘        └─────+────┘│ └─────┘  │  controller  │      └──+──┘
    │9. push into socket recv que        (4)|     │          └──────────────┘  ┌──────┴───────┐
    └───────────────────────────────────────┼─────┘                            │ hardware irq │
                   4. DMA frames to mem.    |   [PCIe]                         └──────+───────┘
────────────────────────────────────────────|─────────────────────────────────────────|──────────
               3. packets send to nic    ┌──┴──┐  5. issue hardware irq notify cpu    |
               ------------------------->│ nic ├--------------------------------------┘
                                         └─────┘
switch the focus window when using gdb -tui
-------------------------------------------
# method-1
press keys 'ctrl+x; o'   # cycle through windows

# method-2
$ focus next
```
{% endraw %}

<hr>

### # Makefile to build this app
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
	$(CXX) $(CFLAGS) -o $@ $^ -lncurses   # can also use '-lncursesw'
.cpp.o:
	$(CXX) $(CFLAGS) -c $< -o $@

.PHONY:clean all
clean:
	rm -f *.o $(OUT) $(ROOTOBJ) $(SUBOBJ)
```

