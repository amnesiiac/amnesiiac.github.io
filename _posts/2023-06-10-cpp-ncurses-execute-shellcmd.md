---
layout: post
title: "tui menu supports shell cmd execution (c++ ncurses)"
author: "melon"
date: 2023-06-10 11:00
categories: "2023"
tags:
  - cpp
  - ncurses
---

### # code 
```cpp
#include <ncurses.h>
#include <cstdlib>
#include <cstring>
#include <string>

char* execute_command(const char* command){
    char buffer[128];
    char* result = static_cast<char*>(malloc(1));
    result[0] = '\0';
    FILE* pipe = popen(command, "r");
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

int main(){
    char command[256];

    // initialize ncurses
    initscr();
    cbreak();
    echo();

    bool quit = false;

    while(!quit){
        // print initial message
        printw("Enter a key (or 'quit' to exit):\n");
        // create an input field
        getstr(command);
        // check if user wants to quit
        if(strcmp(command, "q") == 0){
            quit = true;
            continue;
        }
        // check if the key exists in the dictionary
        // execute the corresponding command
        char* output = execute_command(command);
        // print the output
        printw("\nOutput:\n");
        printw("%s", output);
        // free memory
        free(output);

        // wait for user input to continue or quit
        printw("\nPress any key to continue or enter 'quit' to exit.\n");
        int ch = getch();
        if(ch == 'q' || ch == 'Q')
            quit = true;
        // clear the screen
        clear();
    }

    // end ncurses
    endwin();

    return 0;
}
```
