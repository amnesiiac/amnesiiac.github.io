---
layout: post
title: "tui menu supports key scrolling (c++ ncurses)"
author: "melon"
date: 2023-06-12 18:00
categories: "2023"
tags:
  - cpp
  - ncurses
---

### # TUI program ouput
```txt
    ┌────────────────────────────────────────────────────────────────────┐
    │                                                                    │
    │                                                                    │
    │ * Choice 7     Choice 8     Choice 9                               │
    │   Choice 10    Choice 11    Choice 12                              │
    │   Choice 13    Choice 14    Choice 15                              │
    │   Choice 16    Choice 17    Choice 18                              │
    │   Choice 19    Choice 20    Exit                                   │
    │                                                                    │
    └────────────────────────────────────────────────────────────────────┘

Use PageUp and PageDown to scroll
Use Arrow Keys to navigate (F1 to Exit)
```

<hr>

### # code
```cpp
#include <cstring>
#include <cstdlib>
#include <ncurses.h>
#include <menu.h>

#define ARRAY_SIZE(a) (sizeof(a) / sizeof(a[0]))
#define CTRLD 	4

char *choices[] = {
    "Choice 1", "Choice 2", "Choice 3", "Choice 4", "Choice 5",
    "Choice 6", "Choice 7", "Choice 8", "Choice 9", "Choice 10",
    "Choice 11", "Choice 12", "Choice 13", "Choice 14", "Choice 15",
    "Choice 16", "Choice 17", "Choice 18", "Choice 19", "Choice 20",
    "Exit",
    (char *)NULL,
};

int main(){
    ITEM **my_items;
    int c;				
    MENU *my_menu;
    WINDOW *my_menu_win;
    int n_choices, i;

    initscr(); // initialize curses
    start_color(); // enable color pair preset
    cbreak();
    noecho();
    keypad(stdscr, TRUE);

    init_pair(1, COLOR_RED, COLOR_BLACK); // color preset
    init_pair(2, COLOR_CYAN, COLOR_BLACK);

    // create items
    n_choices = ARRAY_SIZE(choices);
    my_items = (ITEM **)calloc(n_choices, sizeof(ITEM *));
    for(i = 0; i < n_choices; ++i){
        my_items[i] = new_item(choices[i], choices[i]);
    }

    // create menu
    my_menu = new_menu((ITEM **)my_items);

    // set menu option not to show the description
    menu_opts_off(my_menu, O_SHOWDESC);

    // create the window to be associated with the menu
    my_menu_win = newwin(10, 70, 4, 4);
    keypad(my_menu_win, TRUE);

    // set main window and sub window
    set_menu_win(my_menu, my_menu_win);
    set_menu_sub(my_menu, derwin(my_menu_win, 6, 68, 3, 1));
    set_menu_format(my_menu, 5, 3);
    set_menu_mark(my_menu, " * ");

    // print a border around the main window and print a title
    box(my_menu_win, 0, 0);

    attron(COLOR_PAIR(2)); // set color preset on
    mvprintw(LINES - 3, 0, "Use PageUp and PageDown to scroll");
    mvprintw(LINES - 2, 0, "Use Arrow Keys to navigate (F1 to Exit)");
    attroff(COLOR_PAIR(2)); // set color preset off
    refresh();

    // post the menu
    post_menu(my_menu);
    wrefresh(my_menu_win);

    while((c = wgetch(my_menu_win)) != KEY_F(1)){
        switch(c){
            case KEY_DOWN: // down
                menu_driver(my_menu, REQ_DOWN_ITEM);
                break;
            case KEY_UP: // up
                menu_driver(my_menu, REQ_UP_ITEM);
                break;
            case KEY_LEFT: // left
                menu_driver(my_menu, REQ_LEFT_ITEM);
                break;
            case KEY_RIGHT: // right
                menu_driver(my_menu, REQ_RIGHT_ITEM);
                break;
            case KEY_NPAGE: // next page (scroll)
                menu_driver(my_menu, REQ_SCR_DPAGE);
                break;
            case KEY_PPAGE: // pre page (scroll)
                menu_driver(my_menu, REQ_SCR_UPAGE);
                break;
        }
        wrefresh(my_menu_win);
    }	

    // unpost and free all the memory taken up
    unpost_menu(my_menu);
    free_menu(my_menu);
    for(i = 0; i < n_choices; ++i){
        free_item(my_items[i]);
    }
    endwin();
}
```

<hr>

### # Makefile to build this app
remember to use -lncurses to enable link to the ncurse lib.
```makefile
CXX=g++
CFLAGS=-g -Wall -std=c++17 -pthread -w
OUT=out

SUBDIR=$(shell ls -d */)

ROOTSRC=$(wildcard *.cpp)
ROOTOBJ=$(ROOTSRC:%.cpp=%.o)

SUBSRC=$(shell find $(SUBDIR) -name '*.cpp')
SUBOBJ=$(SUBSRC:%.cpp=%.o)

$(OUT):$(ROOTOBJ) $(SUBOBJ)
        $(CXX) $(CFLAGS) -o $@ $^ -lncurses -lmenu
.cpp.o:
        $(CXX) $(CFLAGS) -c $< -o $@

.PHONY:clean all
clean:
        rm -f *.o $(OUT) $(ROOTOBJ) $(SUBOBJ)
```
