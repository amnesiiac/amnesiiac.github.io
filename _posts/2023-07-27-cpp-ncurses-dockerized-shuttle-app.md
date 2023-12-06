---
layout: post
title: "dockerized shuttle app (c++ ncurses)"
author: "melon"
date: 2023-07-27 07:53
categories: "2023"
tags:
  - cpp
  - ncurses
---

<details>
    <summary><b>preview of shuttle usages</b></summary>
    <center><img src="/assets/images/2023/shuttle/shuttle.gif" width="100%"></center>
</details>


{% raw %}
### # dockerfile for frozen env to compile & run shuttle app
```text
# alpine
FROM anthonyzou/alpine-build-essentials

RUN apk update && \
    apk add ncurses-dev
```
make the docker image by:
```text
docker build -t shuttle .
```
commands to build app and run:
```text
# pwd is the current project location
docker run -it --rm -v ${PWD}:/root/shuttle/ -w /root/shuttle shuttle make        # build the binary
docker run -it --rm -v ${PWD}:/root/shuttle/ -w /root/shuttle shuttle ./out       # run the app
docker run -it --rm -v ${PWD}:/root/shuttle/ -w /root/shuttle shuttle make clean  # clean
```

<hr>

### # build up my own env base
inspect the above image base:
```text
$ docker pull anthonyzou/alpine-build-essentials
Using default tag: latest
latest: Pulling from anthonyzou/alpine-build-essentials
5843afab3874: Already exists
91de67d95162: Already exists
Digest: sha256:0f1a97d23690f00bd607ab0e5d5bf91e521d57aa898f31470622ec5cae3904bd
Status: Downloaded newer image for anthonyzou/alpine-build-essentials:latest
docker.io/anthonyzou/alpine-build-essentials:latest
```
check each stacked layer infomation:
```text
$ docker history anthonyzou/alpine-build-essentials
IMAGE             CREATED      CREATED BY                                             SIZE   COMMENT
sha256:f4a660...  2 years ago  RUN /bin/sh -c apk --update add gcc make g++ zlib-dev  190MB  buildkit.dockerfile.v0
<missing>         2 years ago  /bin/sh -c #(nop) CMD ["/bin/sh"]                      0B
<missing>         2 years ago  /bin/sh -c #(nop) ADD file:f2783... in /               5.6MB
```
todo: try using the cmd exposed in layer 1 for built up my own env base.

<hr>

### # project dir strcture
```text
.
├── Dockerfile      # env
├── Makefile        # make
├── desc            # desc files to show when user press tab inside app
│   └── test        # text content with graphic elements supported
├── docs
├── form.h          # search form to enable 'quick selection' by user input (todo globaly)
├── jaro.h          # algorithm to measure the string distance
├── leven.h         # algorithm to measure the string distance
├── main.cpp        # main loop
├── menuitem.h      # json-like data holder to construct menu
├── popup.h         # load desc file and enable display and scroll
├── readme          # todos
└── utils.h

2 directories, 11 files
```

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
	$(CXX) $(CFLAGS) -o $@ $^ -lncurses -lmenu -lform
.cpp.o:
	$(CXX) $(CFLAGS) -c $< -o $@

.PHONY:clean all
clean:
	rm -f *.o $(OUT) $(ROOTOBJ) $(SUBOBJ)
```

<hr>

### # form.h: enable user input for quick selection
```text
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

#include "utils.h"


// static WINDOW *win_body, *win_form;

#define FORMWIDTH 80
#define FORMHEIGHT 1
#define FORMSTART_X 2  // for alignment with main window items
#define FORMSTART_Y 1


static void driver(WINDOW* win, FORM* form, FIELD* fields[3], int ch, string& res){
    int i;
    switch(ch){
        case 10: // enter
            // sync current field buffer with the value displayed
            form_driver(form, REQ_NEXT_FIELD); // refresh cur field buffer
            form_driver(form, REQ_PREV_FIELD); // go back
            refresh();
            res = string(trim_whitespaces(field_buffer(fields[1], 0)));
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
    wrefresh(win);
}

// create subwin (form) inherited from menu_win
string formloop(WINDOW* menu_win){
    // win of form -> menu_win 
    WINDOW* win_form = derwin(menu_win, FORMHEIGHT, FORMWIDTH, FORMSTART_Y, FORMSTART_X);
    assert(win_form != NULL);
    // box(win_form, 0, 0);
    
    // fields
    FIELD* fields[3];
    fields[0] = new_field(1, 10, 0, 0, 0, 0);   // field_height, field_width, pos_y, pos_x
    fields[1] = new_field(1, 40, 0, 8, 0, 0);
    fields[2] = NULL;
    assert(fields[0] != NULL && fields[1] != NULL);
    // set buffer names
    set_field_buffer(fields[0], 0, "SEARCH:");
    set_field_buffer(fields[1], 0, "");
    // set field options
    set_field_opts(fields[0], O_VISIBLE | O_PUBLIC | O_AUTOSKIP);
    set_field_opts(fields[1], O_VISIBLE | O_PUBLIC | O_EDIT | O_ACTIVE);

    // form
    FORM* form = new_form(fields);
    assert(form != NULL);
    set_form_win(form, win_form);
    // set_form_sub(form, derwin(win_form, FORMHEIGHT, FORMWIDTH, FORMSTART_Y, FORMSTART_X)); // bugs
    set_form_sub(form, win_form);
    post_form(form);

    // refresh();
    wrefresh(win_form);

    // main loop
    int ch;
    string res;
    while((ch=getch()) != 27){ // ESC
        driver(win_form, form, fields, ch, res);
        if(ch == 10){ // the second enter will break while
            break;
        }
    }

    // cleanup (todo: make it a hook when tui abnormal exit)
    unpost_form(form);
    free_form(form);
    free_field(fields[0]);
    free_field(fields[1]);
    free_field(fields[2]);
    delwin(win_form);
    // endwin(); // win_form is subwin of menu_win, if endwin, will exit all

    return res;
}

#endif
```

<hr>

### # jaro.h: using jarowinkler algorithm for string similarity computation
```text
#ifndef JAROWINKLER_H_
#define JAROWINKLER_H_

// ref: https://github.com/TriviaMarketing/Jaro-Winkler/tree/master

#include <string>
#include <algorithm>

const double JARO_WEIGHT_STRING_A(1.0/3.0);
const double JARO_WEIGHT_STRING_B(1.0/3.0);
const double JARO_WEIGHT_TRANSPOSITIONS(1.0/3.0);

const unsigned long int JARO_WINKLER_PREFIX_SIZE(4);
const double JARO_WINKLER_SCALING_FACTOR(0.1);
const double JARO_WINKLER_BOOST_THRESHOLD(0.7);

// declaration
double jaroDistance(const std::string& a, const std::string& b);
double jaroWinklerDistance(const std::string&a, const std::string& b);

// definition
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
        for(int index(0), indexEnd(std::min(std::min(a.size(), b.size()), JARO_WINKLER_PREFIX_SIZE)); 
            index<indexEnd; ++index){
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

#endif // JAROWINKLER_HPP
```

<hr>

### # leven.h: levenshtein algorithm for string similarity computation
```text
#ifndef LEVEN_H_
#define LEVEN_H_

// ref: https://github.com/guilhermeagostinelli/levenshtein

#include <iostream>
#include <string.h>
#include <math.h>
#include <algorithm>

// declaration
int levenshteinDist(string word1, string word2);

// definition
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

    // sets the first row and the first column of the verification matrix with the numerical order 
    // from 0 to the length of each word.
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
            // 0 means no modification (i.e. equal letters) and 1 means that a modification is needed.
            int cost = (word2[j-1] == word1[i-1]) ? 0 : 1;

            // sets the current position of the matrix as the minimum value between 
            // a (deletion), b (insertion) and c (substitution).
            // a = the upper adjacent value plus 1: verif[i-1][j] + 1
            // b = the left adjacent value plus 1: verif[i][j-1] + 1
            // c = the upper left adjacent value plus the modification cost: verif[i-1][j-1] + cost
            verif[i][j] = std::min(
                std::min(verif[i-1][j] + 1, verif[i][j-1] + 1),
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

### # main.cpp: app main loop
```text
#include <cstring>
#include <stdio.h>
#include <ncurses.h>
#include <iostream>
#include <locale.h>   // setlocal

#include "menuitem.h"
#include "form.h"     // subwin enable quick selection according to user input
#include "popup.h"    // new popup win that show related file content in man dir
#include "leven.h"    // levenshtein distance
#include "jaro.h"     // jaro-winkler distance: better performance when matching special chars (enabled)

// std
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
#define MENU_X 2          // menu start x
#define MENU_Y 2          // menu start y
#define MENU_INTERVAL 2   // interval between prev/next layer of menus

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
    int pos1 = str.find_first_of('#');
    int pos2 = str.find_last_of('#') - pos1 - 1;
    if(pos1!=string::npos && pos2!=string::npos){
        str.erase(0, pos1+1);
        str.erase(pos2);
        return str;
    }
    return "";
}

// return the filename(man, examples...) if existed
inline string get_file(string str){
    int pos = str.find_last_of('#');
    if(pos != string::npos){
        str.erase(0, pos+1);
        return str;
    }
    return "";
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
    // libc use locale from system default
    setlocale(LC_ALL, "");

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
    double jaroDis = 0;             // jaro-winkler distance
    // int minDis = 1000;              // min l dis
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
            // press '/' -> print a form, ask for user input, and use regex to highlight the best match item
            case 47:
                // using form return a string
                ret = formloop(menu_win);
                // find the best match by levendis
                for(int i=0; i<res[active_layer].size(); ++i){ // for each item in menu_win
                    // levenDis = levenshteinDist(get_head(res[active_layer][i]), ret);
                    // minDisIdx = levenDis<minDis? i : minDisIdx;
                    // minDis = std::min(levenDis, minDis);
                    jaroDis = 1 - jaroWinklerDistance(get_head(res[active_layer][i]), ret);
                    minDisIdx = jaroDis<minDis? i : minDisIdx;
                    minDis = std::min(jaroDis, minDis);
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
            case 9: // tab -> print text on popup window
                showcase(menu_win, get_file(res[active_layer][active_menu[active_layer]]));
                print_menu(menu_win, x, y, active_layer, active_menu, res);
                // choice = true;
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

### # menuitem.h: file for menu data (abbreviated version)
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
    // Create the menu structure
    MenuItem *menu_ptr = new MenuItem;;
    menu_ptr->name = "Main Menu";
    menu_ptr->submenu = {
        {"option4", {{"option4.1", {{"option4.1.1", {{"option4.1.1.1#the command#descfile"},
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

<hr>

### # popup.h: load file under desc w.r.t menuitem definition 
```text
#ifndef POPUP_H_
#define POPUP_H_

#include <unistd.h>   // for sleep 1
#include <iostream> 
#include <fstream>
#include <string>
#include <wchar.h>    // wchar type judge (alphabet, space ...)

#include "utils.h"

// using std::min;

#define POPUPWIDTH 118   // main win-2
#define POPUPHEIGHT 28   // main win-2
#define POPUPSTART_Y 2
#define POPUPSTART_X 2

// write to screen
void show(WINDOW* win, const char* file, int& rowcount, int& startrow);

// create subwin (form) inherited from menu_win
void showcase(WINDOW* menu_win, string filename){
    // derwin: the content in parent win will be kept; newwin: overwrite the 'parent' win
    WINDOW* win_pop = newwin(POPUPHEIGHT, POPUPWIDTH, POPUPSTART_Y, POPUPSTART_X);
    assert(win_pop != NULL);
    // box(win_pop, 0, 0);
    
    refresh();
    wrefresh(win_pop);

    int ch; 
    int rowcount = 0; int startrow = 0;
    bool stop = false;
    string desc = "/root/shuttle/desc/" + filename; // absolute path for using as symbolic tool
    // string desc = string(getrealpath(".")) + "/desc/" + filename;

    // show(win_pop, getrealpath(desc.c_str()), rowcount, startrow); // init content & compute rowcount
    show(win_pop, desc.c_str(), rowcount, startrow); // init content & compute rowcount
    while(1){
        ch = wgetch(menu_win);
        switch(ch){
            case 9:  // tab
                stop = true;
                break;
            case KEY_DOWN:
                if(startrow!=0 && rowcount-startrow<=POPUPHEIGHT){ // disable keydown when no content behind
                    break;
                }
                if(rowcount > POPUPHEIGHT){ // to be checked
                    startrow++; 
                    // wclear(win_pop); // window waiting for next refresh to redraw from scratch (win flickering)
                    werase(win_pop); // just clear the existed content (smoothing between redraws)
                    show(win_pop, getrealpath(desc.c_str()), rowcount, startrow);
                }
                break;
            case KEY_UP:
                if(startrow > 0){
                    startrow--;
                    // wclear(win_pop); // clear -> cause win flickering
                    werase(win_pop);
                    show(win_pop, getrealpath(desc.c_str()), rowcount, startrow);
                }
                break;
            default: // other keys
                break;
        }
        if(stop){
            break;
        }
    }

    delwin(win_pop);
    // endwin();
}

void show(WINDOW* win, const char* file, int& rowcount, int& startrow){
    FILE* fp;
    wchar_t c;
    fp = fopen(file, "r") ; // opening an existing file
    if(fp == NULL){ // show nothing
        return;
    }
    int row = 0; int col = 0; // win start pos
    int currow = 0;          // cursor row num
    // print wchar by wchar
    while(1){
        // mvwprintw(win, row, col, "%s", file); break; // for test file path
        c = getwc(fp); // get wchar from file stream
        if(c == EOF){  // end of file
            break;
        }
        // deal with special tui-elements wchar_t
        if(c == L'─'){
            if(currow >= startrow){
                mvwaddch(win, row, col, ACS_HLINE);
                col++;
            }
        }
        else if(c == L'┌'){
            if(currow >= startrow){
                mvwaddch(win, row, col, ACS_ULCORNER);
                col++;
            }
        }
        else if(c == L'└'){
            if(currow >= startrow){
                mvwaddch(win, row, col, ACS_LLCORNER);
                col++;
            }
        }
        else if(c == L'┘'){
            if(currow >= startrow){
                mvwaddch(win, row, col, ACS_LRCORNER);
                col++;
            }
        }
        else if(c == L'┐'){
            if(currow >= startrow){
                mvwaddch(win, row, col, ACS_URCORNER);
                col++;
            }
        }
        else if(c == L'│'){
            if(currow >= startrow){
                mvwaddch(win, row, col, ACS_VLINE);
                col++;
            }
        }
        else if(c == L'├'){
            if(currow >= startrow){
                mvwaddch(win, row, col, ACS_LTEE);
                col++;
            }
        }
        else if(c == L'├'){
            if(currow >= startrow){
                mvwaddch(win, row, col, ACS_LTEE);
                col++;
            }
        }
        else if(c == L'┤'){
            if(currow >= startrow){
                mvwaddch(win, row, col, ACS_RTEE);
                col++;
            }
        }
        else if(c == L'┴'){
            if(currow >= startrow){
                mvwaddch(win, row, col, ACS_BTEE);
                col++;
            }
        }
        else if(c == L'┬'){
            if(currow >= startrow){
                mvwaddch(win, row, col, ACS_TTEE);
                col++;
            }
        }
        else if(c == L'┼'){
            if(currow >= startrow){
                mvwaddch(win, row, col, ACS_PLUS);
                col++;
            }
        }
        else if(c == '\n'){ // newline
            currow++;
            if(currow >= startrow){ // if need to display the line after \n
                if(currow==startrow && startrow!=0){ // if ...
                    col=0;
                }
                else if(currow==startrow && startrow==0){
                    row++; col=0;
                    mvwprintw(win, row, col, "%c", c);
                }
                else{
                    row++; col=0;
                    mvwprintw(win, row, col, "%c", c);
                }
            }
        }
        else{ // any chars
            if(currow >= startrow){
                mvwprintw(win, row, col, "%c", c); // 0,0 -> 0,1
                col++; // move
            }
        }
    }
    if(startrow == 0){ // record row of entire file when startrow=0
        rowcount = row;
    }
    wrefresh(win);
    fclose(fp); // close
}

#endif
```

<hr>

### # utiles.h
```text
#ifndef UTILS_H_
#define UTILS_H_

// declarations
static char* trim_whitespaces(char *str);
char* getrealpath(const char* symlink, size_t len=100);

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

// #include <stdlib.h>
char* getrealpath(const char* symlink, size_t len){
    char abs_path[len];
    char actual[len+1];
    return realpath(symlink, NULL);
}

#endif
```
{% endraw %}
