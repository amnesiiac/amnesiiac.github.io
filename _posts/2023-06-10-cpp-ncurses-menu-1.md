---
layout: post
title: "tui menu supports key up/down (c++ ncurses)"
author: "melon"
date: 2023-06-10 10:10
categories: "2023"
tags:
  - cpp
  - ncurses
---

### # TUI program ouput
```txt
Use arrow keys to go up and down, Press enter to select a choice
  ┌────────────────────────────┐
  │                            │
  │ Choice 1                   │
  │ Choice 2                   │
  │ Choice 3                   │
  │ Choice 4                   │
  │ Exit                       │
  │                            │
  │                            │
  └────────────────────────────┘
```

<hr>

### # code
```cpp
#include <stdio.h>
#include <ncurses.h>

// init menu win w/h
#define WIDTH 30
#define HEIGHT 10

// init the (x,y) of the menu win
int startx = 0;
int starty = 0;

char *choices[] = {"Choice 1", "Choice 2", "Choice 3", "Choice 4", "Exit",};

int n_choices = sizeof(choices) / sizeof(char *);

void print_menu(WINDOW *menu_win, int highlight){
    int x, y, i;
    x = 2; y = 2;
    box(menu_win, 0, 0);
    for(i = 0; i < n_choices; ++i){ // for each item in menu_win
        if(highlight == i + 1){ // active item -> highlight
            wattron(menu_win, A_REVERSE);
            mvwprintw(menu_win, y, x, "%s", choices[i]);
            wattroff(menu_win, A_REVERSE);
        }
        else{ // no active item
            mvwprintw(menu_win, y, x, "%s", choices[i]);
        }
        ++y;
    }
    wrefresh(menu_win);
}

int main(){
    WINDOW *menu_win;
    int highlight = 1;
    int choice = 0;
    int c;

    initscr();
    clear();
    noecho();
    cbreak();	/* Line buffering disabled. pass on everything */

    startx = 2; starty = 1; // set the start (x,y) of menu win
        
    menu_win = newwin(HEIGHT, WIDTH, starty, startx);
    keypad(menu_win, TRUE);
    mvprintw(0, 0, "Use arrow keys to go up and down, Press enter to select a choice");
    refresh();
    print_menu(menu_win, highlight); // print the init menu win
    while(1){ // keep refreshing the menu win
        c = wgetch(menu_win);
        switch(c){
            case KEY_UP:
                if(highlight == 1){
                    highlight = n_choices;
                }
                else{
                    --highlight;
                }
                break;
            case KEY_DOWN:
                if(highlight == n_choices){
                    highlight = 1;
                }
                else{
                    ++highlight;
                }
                break;
            case 10: // if user press 'enter'
                choice = highlight;
                break;
            default: // if user press other key
                mvprintw(24, 0, "Charcter pressed is = %3d Hopefully it can be printed as '%c'", c, c);
                refresh();
                break;
        }
        print_menu(menu_win, highlight); // refresh the win
        if(choice != 0){ // user choice to get out of infinite loop
            break;
        }
    }
    mvprintw(23, 0, "You chose choice %d with choice string %s\n", choice, choices[choice - 1]);
    clrtoeol();
    refresh();
    endwin();
    return 0;
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
        $(CXX) $(CFLAGS) -o $@ $^ -lncurses
.cpp.o:
        $(CXX) $(CFLAGS) -c $< -o $@

.PHONY:clean all
clean:
        rm -f *.o $(OUT) $(ROOTOBJ) $(SUBOBJ)
```
