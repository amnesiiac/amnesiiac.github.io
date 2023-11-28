---
layout: post
title: "tui menu with quick selection (c++ ncurses)"
author: "melon"
date: 2023-07-14 22:12
categories: "2023"
tags:
  - cpp
  - ncurses
---

### # TUI program ouput
```txt
Arrow keys to go up/down. Press enter to select.
 ┌──────────────────────────────────────────────────────────────────────────────┐
 │ search: no backup                                                            │
 │ mac +                  hg help +      update workdir or switch revisions     │
 │ transhell              hg head +      discard changes in workdir(no backup)  │
 │ blog +                 hg diff +                                             │
 │ latex +                hg annotate +                                         │
 │ gnu +                  hg commit +                                           │
 │ git +                  hg revert +                                           │
 │ mercurial +            hg pull +                                             │
 │ ps +                   hg update +                                           │
 │ pstree +               hg strip +                                            │
 │ ip +                   hg export+                                            │
 │ iptables +             hg import +                                           │
 │ brctl +                hg rebase +                                           │
 │ bridge +               hg backout +                                          │
 │ ethtool +              hg status +                                           │
 │ tcpdump +              hg log +                                              │
 │ docker +               hg tag +                                              │
 │ file/dir operations +  hg in/out +                                           │
 │ nvim +                                                                       │
 │ tmux +                                                                       │
 │ option4                                                                      │
 │ option5                                                                      │
 │ option6                                                                      │
 │                                                                              │
 └──────────────────────────────────────────────────────────────────────────────┘
```

<hr>

{% raw %}

### # mainloop: main.cpp
```cpp
#include <cstring>
#include <stdio.h>
#include <ncurses.h>
#include <iostream>

#include "menuitem.h"
#include "form.h"
#include "leven.h"  // levenshtein distance
#include "jaro.h"   // jaro-winkler distance: better performance in fuzzy string match in this app (enabled)

using std::vector;
using std::string;
using std::map;
using std::max;
using std::cout;
using std::endl;

#define WINWIDTH 120
#define WINHEIGHT 30
#define WINSTART_X 1
#define WINSTART_Y 1
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

// dump "layer -> menus" info to result (only available when memu layer changes)
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
    map<int, vector<string>> res;   // layer -> vector<int> menu names
    int active_layer = 1;           // active layer
    vector<int> active_menu(10, 0); // maximum 10 layer
    int active_layer_menu_num;      // num of items of each layer
    int x=MENU_X, y=MENU_Y;         // menu win pos
    int c;                          // user input char
    bool choice=false;              // user enter choice
    int old_win_pos=x;              // record last operating win x pos
    string head;                    // chosen head name
    string cmd;                     // chosen cmd
    string ret;                     // form return string
    // int levenDis = 0;               // levenshtein distance
    // int minDis = 1000;              // min l dis
    double jaroDis = 0;             // jaro-winkler distance
    double minDis = 1000;
    int minDisIdx = 0;              // menu item index has min l dis

    // init ncurse
    initscr();
    clear();
    noecho();
    cbreak();

    // init ncurse menu win
    WINDOW *menu_win;
    menu_win = newwin(WINHEIGHT, WINWIDTH, WINSTART_Y, WINSTART_X);
    keypad(menu_win, TRUE);
    mvprintw(0, 0, "Arrow keys to go up/down. Press enter to select.");
    refresh(); // refresh all

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
                ++active_layer; active_menu[active_layer]=0; // refresh highlight
                res.clear();
                getlayout(res, ptr, startlayer, active_layer, active_menu); // refresh res
                if(res[active_layer].size()!=0){ // check if current highlight has submenu
                    x=old_win_pos+get_menu_width(res[active_layer-1])+MENU_INTERVAL; old_win_pos=x;
                }
                else{ // do nothing show the same (restore the condition)
                    active_layer--; // restore layer
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
                            x=layer==1? old_win_pos:old_win_pos+get_menu_width(res[layer-1])+MENU_INTERVAL;
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
            // press '/' -> print a form on top of window, ask for user input, 
            // and use regex to highlight the best match item
            case 47: 
                // using form return a string
                ret = formloop(menu_win);
                // find the best match by levendis
                for(int i=0; i<res[active_layer].size(); ++i){ // for each item in menu_win
                    // levenDis = levenshteinDist(get_head(res[active_layer][i]), ret);
                    // minDisIdx = levenDis<minDis? i : minDisIdx;
                    // minDis = min(levenDis, minDis);
                    jaroDis = 1 - jaroWinklerDistance(get_head(res[active_layer][i]), ret);
                    minDisIdx = jaroDis<minDis? i : minDisIdx;
                    minDis = min(jaroDis, minDis);
                }
                // refresh cur hi 
                if(minDisIdx != active_menu[active_layer]){
                    active_menu[active_layer] = minDisIdx;
                    // levenDis = 0; minDis = 1000; minDisIdx = 0; // reset
                    jaroDis = 0; minDis = 1000; minDisIdx = 0; // reset
                }
                // update layout info and print
                print_menu(menu_win, x, y, active_layer, active_menu, res);
                break;
            default: // other keys -> refresh the menu_win
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

<hr>

### # quick selection using (jaro-winkler distance): jaro.h
```cpp
#ifndef JARO_H_
#define JARO_H_

#include <string>
#include <algorithm>

// ref: https://github.com/TriviaMarketing/Jaro-Winkler/tree/master
const double JARO_WEIGHT_STRING_A(1.0/3.0);
const double JARO_WEIGHT_STRING_B(1.0/3.0);
const double JARO_WEIGHT_TRANSPOSITIONS(1.0/3.0);

const unsigned long int JARO_WINKLER_PREFIX_SIZE(4);
const double JARO_WINKLER_SCALING_FACTOR(0.1);
const double JARO_WINKLER_BOOST_THRESHOLD(0.7);


double jaroDistance(const std::string& a, const std::string& b);
double jaroWinklerDistance(const std::string&a, const std::string& b);

double jaroDistance(const std::string& a, const std::string& b){
    // register strings length
    int aLength(a.size());
    int bLength(b.size());
    
    // if one string has null length, we return 0.
    if(aLength == 0 || bLength == 0){
        return 0.0;
    }
    
    // calculate max length range
    int maxRange(std::max(0, std::max(aLength, bLength)/2-1));
    
    // creates 2 vectors of integers
    std::vector<bool> aMatch(aLength, false);
    std::vector<bool> bMatch(bLength, false);
    
    // calculate matching characters
    int matchingCharacters(0);
    for(int aIndex(0); aIndex<aLength; ++aIndex){
        // calculate window test limits (limit inferior to 0 and superior to bLength)
        int minIndex(std::max(aIndex-maxRange, 0));
        int maxIndex(std::min(aIndex+maxRange+1, bLength));
        
        if(minIndex >= maxIndex){
            // no more common character because we don't have characters in b to test with characters in a
            break;
        }
        
        for(int bIndex(minIndex); bIndex<maxIndex; ++bIndex){
            if (!bMatch.at(bIndex) && a.at(aIndex) == b.at(bIndex)){
                // found some new match
                aMatch[aIndex] = true;
                bMatch[bIndex] = true;
                ++matchingCharacters;
                break;
            }
        }
    }
    
    // if no matching characters, we return 0
    if(matchingCharacters == 0){
        return 0.0;
    }
    
    // calculate character transpositions
    std::vector<int> aPosition(matchingCharacters, 0);
    std::vector<int> bPosition(matchingCharacters, 0);
    
    for(int aIndex(0), positionIndex(0); aIndex<aLength; ++aIndex){
        if(aMatch.at(aIndex)){
            aPosition[positionIndex] = aIndex;
            ++positionIndex;
        }
    }
    
    for(int bIndex(0), positionIndex(0); bIndex<bLength; ++bIndex){
        if(bMatch.at(bIndex)){
            bPosition[positionIndex] = bIndex;
            ++positionIndex;
        }
    }
    
    // counting half-transpositions
    int transpositions(0);
    for(int index(0); index<matchingCharacters; ++index){
        if(a.at(aPosition.at(index)) != b.at(bPosition.at(index))){
            ++transpositions;
        }
    }
    
    // calculate Jaro distance
    return (
        JARO_WEIGHT_STRING_A * matchingCharacters / aLength +
        JARO_WEIGHT_STRING_B * matchingCharacters / bLength +
        JARO_WEIGHT_TRANSPOSITIONS * (matchingCharacters-transpositions/2) / matchingCharacters
    );
}

double jaroWinklerDistance(const std::string&a, const std::string& b){
    // calculate Jaro distance
    double distance(jaroDistance(a, b));
    
    if (distance > JARO_WINKLER_BOOST_THRESHOLD){
        // calculate common string prefix
        int commonPrefix(0);
        for(int index(0), indexEnd(std::min(std::min(a.size(), b.size()), 
            JARO_WINKLER_PREFIX_SIZE)); index<indexEnd; ++index){
            if(a.at(index) == b.at(index)){
                ++commonPrefix;
            }
            else{
                break;
            }
        }
        // calculate Jaro-Winkler distance
        distance += JARO_WINKLER_SCALING_FACTOR * commonPrefix * (1.0-distance);
    }
    return distance;
}

#endif // JARO_H_
```

<hr>

### # quick selection using (levenshtein distance): leven.h
```cpp
#ifndef LEVEN_H_
#define LEVEN_H_

#include <iostream>
#include <string.h>
#include <math.h>
#include <algorithm>

using std::string;
using std::min;
using std::cout;
using std::cin;
using std::endl;


// ref: https://github.com/guilhermeagostinelli/levenshtein
int levenshteinDist(string word1, string word2){
    int size1 = word1.size();
    int size2 = word2.size();
    int verif[size1+1][size2+1]; // verification matrix: 2d arr to store the calculated distance

    // if one of the words has zero length, the distance is equal to the size of the other word.
    if(size1 == 0){
        return size2;
    }
    if(size2 == 0){
        return size1;
    }

    // sets the first row and the first column of the verification matrix 
    // with the numerical order from 0 to the length of each word.
    for(int i=0; i<=size1; i++){
        verif[i][0] = i;
    }
    for(int j=0; j<=size2; j++){
        verif[0][j] = j;
    }

    // verification step / matrix filling.
    for(int i=1; i<=size1; i++){
        for(int j=1; j<=size2; j++){
            // sets modification cost.
            // 0 means no modification (i.e. equal letters) 
            // 1 means that a modification is needed (i.e. unequal letters).
            int cost = (word2[j-1] == word1[i-1]) ? 0 : 1;

            // sets the current position of the matrix as the minimum value 
            // between a (deletion), b (insertion) and c (substitution).
            // a = the upper adjacent value plus 1: verif[i-1][j] + 1
            // b = the left adjacent value plus 1: verif[i][j-1] + 1
            // c = the upper left adjacent value plus the modification cost: verif[i-1][j-1] + cost
            verif[i][j] = min(
                min(verif[i-1][j] + 1, verif[i][j-1] + 1),
                verif[i-1][j-1] + cost
            );
        }
    }

    // the last position of the matrix will contain the Levenshtein distance.
    return verif[size1][size2];
}

#endif
```

<hr>

### # form widget for quick selection input: form.cpp & form.h
##### form.cpp
```cpp
#include "form.h"

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

static void driver(int ch, string& res){
    int i;
    switch(ch){
        case 10: // enter
            // sync current field buffer with the value displayed
            form_driver(form, REQ_NEXT_FIELD); // refresh cur field buffer
            form_driver(form, REQ_PREV_FIELD); // go back
            // move(LINES-3, 2);
            // for(i = 0; fields[i]; i++){ // print label & values in the form
            //     printw("%s", trim_whitespaces(field_buffer(fields[i], 0)));
            //     if(field_opts(fields[i]) & O_ACTIVE){
            //         printw("\"\t");
            //     }
            //     else{
            //         printw(": \"");
            //     }
            // }
            refresh();
            res = string(trim_whitespaces(field_buffer(fields[1], 0)));
            // pos_form_cursor(form);
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

// --- create subwin (form) inherited from menu_win --- //
string formloop(WINDOW* menu_win){
    int ch;

    // sub box window inherited from menu_win
    win_form = derwin(menu_win, FORMHEIGHT, FORMWIDTH, FORMSTART_Y, FORMSTART_X);
    assert(win_form != NULL);
    box(win_form, 0, 0);
    
    fields[0] = new_field(1, 10, 0, 0, 0, 0);   // field_height, field_width, pos_y, pos_x
    fields[1] = new_field(1, 40, 0, 8, 0, 0);
    assert(fields[0] != NULL && fields[1] != NULL);

    // set buffer names
    set_field_buffer(fields[0], 0, "search:");
    set_field_buffer(fields[1], 0, "");

    // set field options
    set_field_opts(fields[0], O_VISIBLE | O_PUBLIC | O_AUTOSKIP);
    set_field_opts(fields[1], O_VISIBLE | O_PUBLIC | O_EDIT | O_ACTIVE);

    // set field style
    // set_field_back(fields[1], A_UNDERLINE);
    // set_field_back(fields[3], A_UNDERLINE);

    // field win (no border)
    form = new_form(fields);
    assert(form != NULL);
    set_form_win(form, win_form);
    set_form_sub(form, derwin(win_form, FORMHEIGHT, FORMWIDTH, FORMSTART_Y, FORMSTART_X));
    post_form(form);

    refresh();
    wrefresh(win_form);

    // main loop
    string res;
    while((ch=getch()) != 27){ // ESC
        driver(ch, res);
    }

    // cleanup (todo: make it a hook when tui abnormal exit)
    unpost_form(form);
    free_form(form);
    free_field(fields[0]);
    free_field(fields[1]);
    delwin(win_form);

    return res;
}
```
##### form.h
```cpp
#ifndef FORM_H_
#define FORM_H_

#include <ncurses.h>
#include <form.h>
#include <assert.h>
#include <string.h>
#include <stdlib.h>
#include <ctype.h>

// c++ string
#include <string>
using std::string;

#include <iostream>
using std::cout;
using std::endl;

static FORM *form;
static FIELD *fields[5];
static WINDOW *win_body, *win_form;

#define FORMWIDTH 80
#define FORMHEIGHT 1
#define FORMSTART_X 2  // for alignment with main window items
#define FORMSTART_Y 1

static char* trim_whitespaces(char *str);

static void driver(int ch, string& res);

string formloop(WINDOW* menu_win);

#endif
```

<hr>

### # menu data static file: menuitem.h
```text
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
        {"option4", {{"option4.1", {{"option4.1.1", {{"option4.1.1.1"},
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

{% endraw %}

<hr>

### # Makefile to build this app
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
	$(CXX) $(CFLAGS) -o $@ $^ -lncurses -lmenu -lform
.cpp.o:
	$(CXX) $(CFLAGS) -c $< -o $@

.PHONY:clean all
clean:
	rm -f *.o $(OUT) $(ROOTOBJ) $(SUBOBJ)
```

<hr>

### # dependencies
```txt
g++ on macos
Apple clang version 11.0.3 (clang-1103.0.32.62)
Target: x86_64-apple-darwin20.6.0
Thread model: posix
InstalledDir: /Library/Developer/CommandLineTools/usr/bin
```
