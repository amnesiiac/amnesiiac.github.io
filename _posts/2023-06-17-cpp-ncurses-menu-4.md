---
layout: post
title: "tui multilayer menu supports execution of shell cmd (c++ ncurses)"
author: "melon"
date: 2023-06-17 16:52
categories: "2023"
tags:
  - cpp
  - ncurses
---

### # TUI program ouput
```txt
Arrow keys to go up/down/right/left. Press enter to select.
 ┌──────────────────────────────────────────────────────────────────────────┐
 │                                                                          │
 │ mac +                  nvim image +     edit nvim dockerfile             │
 │ transhell              nvim commands +  build nvim image                 │
 │ blog +                                  invoke nvim image by script      │
 │ latex +                                                                  │
 │ git +                                                                    │
 │ mercurial +                                                              │
 │ ip +                                                                     │
 │ docker +                                                                 │
 │ file/dir operations +                                                    │
 │ nvim +                                                                   │
 │ tmux +                                                                   │
 │ option4                                                                  │
 │ option5                                                                  │
 │ option6                                                                  │
 │                                                                          │
 └──────────────────────────────────────────────────────────────────────────┘
edit nvim dockerfile: nvim /Users/mac/nvim/Dockerfile
```

<hr>

### # code
{% raw %}
menuitem.h: the header file to init 'json-like' key-value data.
```cpp
#include <vector>
#include <string>
#include <map>

struct MenuItem{
    std::string name;
    std::vector<MenuItem> submenu;
};

// load data into menu data structure => todo: move the data structure to a standalone file
MenuItem* initdata(){
    // create the menu structure
    MenuItem *menu_ptr = new MenuItem;;
    menu_ptr->name = "Main Menu";
    menu_ptr->submenu = {
        {"option1"},
        {"option2"},
        {"option3", {{"option3.1#echo command test 3.1"},{"option3.2#echo command test 3.2"}}},
        {"option4", {{"option4.1", {{"option4.1.1", {{"option4.1.1.1#command test4.1.1.1"},
                                                     {"option4.1.1.2"}}},
                                    {"option4.1.2"}}},
                     {"option4.2"}}},
        {"option5"},
        {"option6", {{"option6.1", {{"option6.1.1", {{"option6.1.1.1",{{"option6.1.1.1.1#echo hi!"},
                                                                       {"option6.1.1.1.2"}}},
                                                     {"option6.1.1.2"}}},
                                    {"option6.1.2"}}},
                     {"option6.2"}}},
    };
    return menu_ptr;
}
```
main.cpp: the ncurse win while loop, recursively process the 'json-like' data and paste the corresponding cmd into system clipboard.
```cpp
#include <stdio.h>
#include <ncurses.h>
#include <iostream>
using namespace std;

#include "menuitem.h"


#define WINWIDTH 120
#define WINHEIGHT 30
#define WINSTART_X 1
#define WINSTART_Y 2
#define MENU_X 2
#define MENU_Y 2
#define MENU_INTERVAL 2


// execute shell cmd: echo "cmd" | pbcopy
char* execute_command(const char *command){
    char buffer[128];
    char *result = static_cast<char*>(malloc(1));
    result[0] = '\0';
    FILE *pipe = popen(command, "r");
    if(pipe == nullptr){
        return result;
    }
    while(fgets(buffer, sizeof(buffer), pipe) != nullptr){
        result = static_cast<char*>(realloc(result, strlen(result) + strlen(buffer) + 1));
        strcat(result, buffer);
    }
    pclose(pipe);
    return result;
}

// return the menu item name
inline string get_head(string str){
    int pos = str.find_first_of('#');
    if(pos != string::npos){
        str.erase(pos);
    }
    return str;
}

// return the menu item cmd
inline string get_cmd(string str){
    int pos = str.find_last_of('#');
    if(pos != string::npos){
        str.erase(0, pos+1);
        return str;
    }
    else{
        return "";
    }
}

// record "layer -> menus" info to result
void getlayout(map<int, vector<string>> &res, MenuItem *ptr, int layer, int limit, 
               vector<int> &highlight){
    if(ptr!=nullptr){
        if(res.find(layer)!=res.end()){
            res[layer].push_back(ptr->name);
        }
        else{
            res[layer] = {ptr->name};
        }
        layer++; // 1 layer deeper
        int menu_no = -1;
        for(auto it : ptr->submenu){
            menu_no++;
            // go to submenu, condition
            if(it.submenu.size()!=0 && layer<limit && menu_no==highlight[layer]){
                getlayout(res, &it, layer, limit, highlight);
            }
            else{ // no submenu 
                if(res.find(layer)!=res.end()){ // already recorded
                    res[layer].push_back(it.name);
                }
                else{
                    res[layer] = {it.name};
                }
            }
        }
        return;
    }
    else{
        return;
    }
}

// print text according to (x,y)
void print_menu(WINDOW *menu_win, int x, int y, int active_layer, vector<int> &highlight, 
                map<int, vector<string>> &res){
    box(menu_win, 0, 0);
    for(int i = 0; i < res[active_layer].size(); ++i){ // for each item in menu_win
        if(highlight[active_layer] == i){ // active item -> highlight
            wattron(menu_win, A_REVERSE);
            mvwprintw(menu_win, y, x, "%s", get_head(res[active_layer][i]).c_str());
            wattroff(menu_win, A_REVERSE);
        }
        else{ // no active item
            mvwprintw(menu_win, y, x, "%s", (char*)get_head(res[active_layer][i]).c_str());
        }
        ++y;
    }
    wrefresh(menu_win);
}

// get max menu name width for each layer
int get_menu_width(vector<string> &menu){
    int len=-1;
    for(auto item:menu){
        len = max(len, (int)get_head(item).length());
    }
    return len;
}

int main(){
    // menu data intake
    MenuItem *ptr = initdata(); 

    // init state
    int startlayer=0;
    map<int, vector<string>> res;    // layer -> vector<int> menu names
    int active_layer = 1;            // active layer 
    vector<int> active_menu(10, 0);  // maximum 10 layer
    int active_layer_menu_num;       // num of items of each layer
    int x=MENU_X, y=MENU_Y;          // menu win pos
    int c;                           // user input char
    bool choice=false;               // user enter choice
    int old_win_pos=x;               // record last operating win x pos
    string head;                     // chosen head name
    string cmd;                      // chosen cmd

    // init ncurse
    initscr();
    clear();
    noecho();
    cbreak();

    // init ncurse menu win
    WINDOW *menu_win;
    menu_win = newwin(WINHEIGHT, WINWIDTH, WINSTART_X, WINSTART_X);
    keypad(menu_win, TRUE);
    mvprintw(0, 0, "Arrow keys to go up/down. Press enter to select.");
    refresh();

    // init layer 1 menu
    getlayout(res, ptr, startlayer, active_layer, active_menu);
    print_menu(menu_win, x, y, active_layer, active_menu, res);

    // tui while loop
    while(1){
        c = wgetch(menu_win);
        switch(c){
            case KEY_UP: // active layer not change, menu highlight item rotated
                active_layer_menu_num = res[active_layer].size(); // compute current layer menu num
                if(active_menu[active_layer] == 0){ // rollback
                    active_menu[active_layer] = active_layer_menu_num-1;
                }
                else{
                    --active_menu[active_layer];
                }
                print_menu(menu_win, x, y, active_layer, active_menu, res);
                break;
            case KEY_DOWN: // active layer not change, menu highlight item rotated 
                active_layer_menu_num = res[active_layer].size();
                if(active_menu[active_layer] == active_layer_menu_num-1){
                    active_menu[active_layer] = 0;
                }
                else{
                    ++active_menu[active_layer];
                }
                print_menu(menu_win, x, y, active_layer, active_menu, res);
                break;
            case KEY_RIGHT: // try active layer+1, has submenu->print, no submenu->restore&print
                ++active_layer;  active_menu[active_layer]=0; // refresh highlight
                res.clear();
                getlayout(res, ptr, startlayer, active_layer, active_menu); // refresh res
                if(res[active_layer].size()!=0){ // check if current highlight has submenu
                    x=old_win_pos+get_menu_width(res[active_layer-1])+MENU_INTERVAL; old_win_pos=x;
                }
                else{ // do nothing show the same (restore the condition)
                    active_layer--;  // restore layer
                    res.clear(); getlayout(res, ptr, startlayer, active_layer, active_menu);
                }
                print_menu(menu_win, x, y, active_layer, active_menu, res);
                break;
            case KEY_LEFT: // go one layer shallower, into pre level active
                if(active_layer<=1){ // no previous menu
                    print_menu(menu_win, x, y, active_layer, active_menu, res);
                }
                else{
                    werase(menu_win); // clear the window
                    old_win_pos=2; // init win pos
                    for(int layer=1; layer<active_layer; layer++){ // print each layer menu left->right
                        res.clear();
                        getlayout(res, ptr, startlayer, layer, active_menu); // refresh res
                        if(res[layer].size()!=0){ // check if current highlight has submenu
                            x=layer==1? old_win_pos:
                                        old_win_pos+get_menu_width(res[layer-1])+MENU_INTERVAL;
                            old_win_pos=x;
                        }
                        print_menu(menu_win, x, y, layer, active_menu, res);
                    }
                    // reset active layer
                    active_layer--;
                    res.clear();
                    getlayout(res, ptr, startlayer, active_layer, active_menu); // refresh res
                }
                break;
            case 10: // press enter -> execute the corresponding shell command
                head = get_head(res[active_layer][active_menu[active_layer]]);
                cmd = get_cmd(res[active_layer][active_menu[active_layer]]);
                print_menu(menu_win, x, y, active_layer, active_menu, res);
                choice = true; // reset choice to break the while loop
                break;
            default:
                print_menu(menu_win, x, y, active_layer, active_menu, res);
                refresh();
                break;
        }
        if(choice){ // user choice to get out of infinite loop
            break;
        }
    }

    // renovate cmd to pbcopy
    string echo = "echo ";
    string quote = "\'"; // forbid shell expansion
    string tr = " | tr -d \'\\n\'";
    string pbcopy = " | pbcopy";
    string cache_cmd = cmd;
    cmd = echo + quote + cmd + quote + tr + pbcopy; 

    // shell cmd execute
    char *output = execute_command(cmd.c_str());
    free(output);

    // clean all & close win
    clrtoeol();
    refresh();
    endwin();

    // output shell prompt    
    cout << head << ": "<< cache_cmd << endl;

    return 0;
}
```
{% endraw %}

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

<hr>

### # reference
https://tldp.org/HOWTO/NCURSES-Programming-HOWTO/index.html
