---
layout: post
title: "tui panel supports multi-win switching (c++ ncurses)"
author: "melon"
date: 2023-07-04 21:36
categories: "2023"
tags:
  - cpp
  - ncurses
---

### # TUI program ouput
```txt
          ┌──────────────────────────────────────┐
          │           Window Number 1            │
          ├──────────────────────────────────────┤
          │      ┌──────────────────────────────────────┐
          │      │           Window Number 2            │
          │      ├──────────────────────────────────────┤
          │      │      ┌──────────────────────────────────────┐
          │      │      │           Window Number 3            │
          │      │      ├──────────────────────────────────────┤
          └──────│      │                                      │
                 │      │                                      │
                 │      │                                      │
                 └──────│                                      │
                        │                                      │
                        │                                      │
                        └──────────────────────────────────────┘
use tab to browse throught the windows (f1 to exit)
```

<hr>

### # code
{% raw %}
```cpp
#include <cstring>  // for strlen() 
#include <panel.h>

#define NLINES 10  // win height
#define NCOLS 40   // win width

void init_wins(WINDOW **wins, int n);
void win_show(WINDOW *win, char *label, int label_color);
void print_in_middle(WINDOW *win, int starty, int startx, int width, char *string, chtype color);

int main(){
    WINDOW *my_wins[3];
    PANEL *my_panels[3];
    PANEL *top;
    int ch;

    // initialize curses
    initscr();
    cbreak();
    noecho();
    keypad(stdscr, TRUE);

    // start_color(); // starting using colors
    // initialize all the colors
    // init_pair(1, COLOR_GREEN, COLOR_WHITE);
    // init_pair(2, COLOR_RED, COLOR_WHITE);
    // init_pair(3, COLOR_BLUE, COLOR_WHITE);
    // init_pair(4, COLOR_CYAN, COLOR_WHITE);

    init_wins(my_wins, 3);

    // attach a panel to each window(bottom-up): the first window is deepest on screen
    my_panels[0] = new_panel(my_wins[0]); // push 0, order: stdscr−0 */
    my_panels[1] = new_panel(my_wins[1]); // push 1, order: stdscr−0−1 */
    my_panels[2] = new_panel(my_wins[2]); // push 2, order: stdscr−0−1−2 */

    // set up the user pointers to the next panel (panel iterate loop)
    set_panel_userptr(my_panels[0], my_panels[1]);
    set_panel_userptr(my_panels[1], my_panels[2]);
    set_panel_userptr(my_panels[2], my_panels[0]);

    // update the win (show init)
    update_panels();

    // start panel updating
    // attron(COLOR_PAIR(4));
    mvprintw(LINES-2, 0, "use tab to browse throught the windows (f1 to exit)");
    // attroff(COLOR_PAIR(4));
    doupdate();
    top = my_panels[2];
    while((ch=getch()) != KEY_F(1)){ // f1 to quit
        switch(ch){
            case 9: // tab to switch panel
                top = (PANEL *)panel_userptr(top);
                top_panel(top);
                break;
        }
        update_panels();
        doupdate(); // ?
    }
    endwin();
    return 0;
}

// init win vector
void init_wins(WINDOW **wins, int n){
    int x, y, i;
    char label[80];
    y = 2;
    x = 10;
    for(i=0; i<n; ++i){
        wins[i] = newwin(NLINES, NCOLS, y, x);
        sprintf(label, "Window Number %d", i+1);
        win_show(wins[i], label, i+1); // show win
        y += 3;
        x += 7;
    }
}

// show the window with border/label
void win_show(WINDOW *win, char *label, int label_color){
    int startx, starty, height, width;
    getbegyx(win, starty, startx);
    getmaxyx(win, height, width);
    // win box
    box(win, 0, 0);
    // win mid line
    mvwaddch(win, 2, 0, ACS_LTEE); // middle line left corner
    mvwhline(win, 2, 1, ACS_HLINE, width-2); // line
    mvwaddch(win, 2, width-1, ACS_RTEE); // right corner
    // print win label
    print_in_middle(win, 1, 0, width, label, COLOR_PAIR(label_color));
}

// print text label in middle
void print_in_middle(WINDOW *win, int starty, int startx, int width, char *string, chtype color){
    int length, x, y;
    float temp;
    if(win == NULL){ // none: show default std win
        win = stdscr;
    }
    getyx(win, y, x); // get current cursor pos
    if(startx != 0){
        x = startx;
    }
    if(starty != 0){
        y = starty;
    }
    if(width == 0){
        width = 80;
    }
    length = strlen(string);
    temp = (width - length)/2;
    x = startx + (int)temp;
    // wattron(win, color);
    mvwprintw(win, y, x, "%s", string);
    // wattroff(win, color);
    refresh();
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
	$(CXX) $(CFLAGS) -o $@ $^ -lpanel -lncurses
.c.o:
	$(CXX) $(CFLAGS) -c $< -o $@

.PHONY:clean all
clean:
	rm -f *.o $(OUT) $(ROOTOBJ) $(SUBOBJ)
```

<hr>
