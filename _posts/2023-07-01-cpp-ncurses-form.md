---
layout: post
title: "tui form supports user-input with callback (c++ ncurses)"
author: "melon"
date: 2023-07-01 17:37
categories: "2023"
tags:
  - cpp
  - ncurses
---

```txt
  ┌──────────────────────────────────────────────────────────────────────────────┐
  │F1 to quit, ENTER to print fields content                                     │
  │┌────────────────────────────────────────────────────────────────────────────┐│
  ││field1         value1                                                       ││
  ││                                                                            ││
  ││field2         value2 hello                                                 ││
  │└────────────────────────────────────────────────────────────────────────────┘│
  └──────────────────────────────────────────────────────────────────────────────┘
  field1: "value1"      field2: "value2 hello"
```

<hr>

### # code
{% raw %}
```text
#include <ncurses.h>
#include <form.h>
#include <assert.h>
#include <string.h>
#include <stdlib.h>
#include <ctype.h>

static FORM *form;
static FIELD *fields[5];
static WINDOW *win_body, *win_form;

#define WINWIDTH 80
#define WINHEIGHT 8 
#define WINSTART_X 1
#define WINSTART_Y 2

// ncurses fill fields blanks with spaces, trim appended spaces to get result
static char* trim_whitespaces(char *str){
    char *end;
    while(isspace(*str)){ // trim leading spaces
        str++;
    }
    if(*str == 0){ // if sll apaces
        return str;
    }
    end = str + strnlen(str, 128) - 1; // trim trailing spaces
    while(end > str && isspace(*end)){
        end--;
    }
    *(end+1) = '\0'; // end
    return str;
}

static void driver(int ch){
    int i;
    switch(ch){
        case 10: // enter
            // sync current field buffer with the value displayed
            form_driver(form, REQ_NEXT_FIELD); // refresh cur field buffer
            form_driver(form, REQ_PREV_FIELD); // go back
            move(LINES-3, 2);
            for(i = 0; fields[i]; i++){ // print label & values in the form
                printw("%s", trim_whitespaces(field_buffer(fields[i], 0)));
                if(field_opts(fields[i]) & O_ACTIVE){
                    printw("\"\t");
                }
                else{
                    printw(": \"");
                }
            }
            refresh();
            pos_form_cursor(form);
            break;
        case KEY_DOWN: // next
            form_driver(form, REQ_NEXT_FIELD);
            form_driver(form, REQ_END_LINE);
            break;
        case KEY_UP: // prev
            form_driver(form, REQ_PREV_FIELD);
            form_driver(form, REQ_END_LINE);
            break;
        case KEY_LEFT:
            form_driver(form, REQ_PREV_CHAR);
            break;
        case KEY_RIGHT:
            form_driver(form, REQ_NEXT_CHAR);
            break;
        case KEY_BACKSPACE: // del char before cursor
        case 127:
            form_driver(form, REQ_DEL_PREV);
            break;
        case KEY_DC: // del char under cursor
            form_driver(form, REQ_DEL_CHAR);
            break;
        default:
            form_driver(form, ch);
            break;
    }
    wrefresh(win_form);
}

int main(){
    int ch;

    initscr();
    noecho();
    cbreak();
    keypad(stdscr, TRUE);

    // main box window
    win_body = newwin(WINHEIGHT, WINWIDTH, WINSTART_X, WINSTART_Y);
    assert(win_body != NULL);
    box(win_body, 0, 0);
    mvwprintw(win_body, 1, 1, "F1 to quit, ENTER to print fields content");

    // sub box window
    win_form = derwin(win_body, WINHEIGHT-3, WINWIDTH-2, 2, 1);
    assert(win_form != NULL);
    box(win_form, 0, 0);
    
    fields[0] = new_field(1, 10, 0, 0, 0, 0);
    fields[1] = new_field(1, 40, 0, 15, 0, 0);
    fields[2] = new_field(1, 10, 2, 0, 0, 0);
    fields[3] = new_field(1, 40, 2, 15, 0, 0);
    fields[4] = NULL;
    assert(fields[0] != NULL && fields[1] != NULL && fields[2] != NULL && fields[3] != NULL);

    // set buffer names
    set_field_buffer(fields[0], 0, "field1");
    set_field_buffer(fields[1], 0, "value1");
    set_field_buffer(fields[2], 0, "field2");
    set_field_buffer(fields[3], 0, "value2");

    // set field options
    set_field_opts(fields[0], O_VISIBLE | O_PUBLIC | O_AUTOSKIP);
    set_field_opts(fields[1], O_VISIBLE | O_PUBLIC | O_EDIT | O_ACTIVE);
    set_field_opts(fields[2], O_VISIBLE | O_PUBLIC | O_AUTOSKIP);
    set_field_opts(fields[3], O_VISIBLE | O_PUBLIC | O_EDIT | O_ACTIVE);

    // set field style
    set_field_back(fields[1], A_UNDERLINE);
    set_field_back(fields[3], A_UNDERLINE);

    // field win (no border)
    form = new_form(fields);
    assert(form != NULL);
    set_form_win(form, win_form);
    set_form_sub(form, derwin(win_form, WINHEIGHT-5, WINWIDTH-4, 1, 1));
    post_form(form);

    refresh();
    wrefresh(win_body);
    wrefresh(win_form);

    // main loop
    while((ch=getch()) != KEY_F(1)){
        driver(ch);
    }

    // cleanup (todo: make it a hook when tui abnormal exit)
    unpost_form(form);
    free_form(form);
    free_field(fields[0]);
    free_field(fields[1]);
    free_field(fields[2]);
    free_field(fields[3]);
    delwin(win_form);
    delwin(win_body);
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
	$(CXX) $(CFLAGS) -o $@ $^ -lform -lncurses
.c.o:
	$(CXX) $(CFLAGS) -c $< -o $@

.PHONY:clean all
clean:
	rm -f *.o $(OUT) $(ROOTOBJ) $(SUBOBJ)
```

<hr>
