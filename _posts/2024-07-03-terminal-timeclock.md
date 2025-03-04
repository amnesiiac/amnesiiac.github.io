---
layout: post
title: "timeclock in terminal (terminal, session, job)"
author: "melon"
date: 2024-07-03 07:11
categories: "2024"
tags:
  - terminal
  - todo
---

some implementations of tui timeclock for linux tty.

get the final workable version of timeclock + crontab to trigger every 5min check for the idle time,
if the idle time is exceed certain value for certain tty, then trigger the foreground timeclock on it.
if the idle time is not exceed the limit, then do nothing.

<hr>

### # txt timeclock
the below script displays current time in red at the top right corner of the terminal every second:

```text
#!/bin/sh

while sleep 1; do
    tput sc;                                  # save the current cursor pos
    tput cup 0 $(($(tput cols)-11));          # move the cursor to the top right corner, 11 col from the right edge
    echo -e "\e[31m$(date +%r)\e[39m";        # -e enables interpretation of backslash escapes
    tput rc;                                  # restore the cursor to the saved pos
done &;
```

in addition to just display time, the script can also be used to avoid ssh reply timeout,
which lead to connection close by peer issue.

the following script can be added into ~/.bashrc to facilitate the shortcut start & stop of the timeclock by
shortcut as typing 'ttt' or 'sss'.

```text
ttt() {                                       # cmd to start up timeclock
    while sleep 1; do
        tput sc;
        tput cup 0 $(($(tput cols)-11));
        echo -e "\e[31m`date +%r`\e[39m";
        tput rc;
    done &
}

sss() {
    kill -sigkill %1                          # cmd to kill the bg timeclock
}

export -f ttt                                 # export file name as executable cmd
export -f sss
```

<hr>

### # a simple tui timeclock (ncurses)
this app can showup on both current tty & other tty, while only active for keyboard stop on current tty;
when displaying on another tty, the keystroke wont take effect to shutdown the program, cause
the app process cannot take precedence over current tty itself, thus the keystock is taken by tty.

```text
// gcc -o out test.c -lncurses
#include <stdio.h>
#include <unistd.h>
#include <fcntl.h>
#include <time.h>
#include <string.h>
#include <ncurses.h>

int timeclock(){
    initscr();                                                         // init screen
    curs_set(0);                                                       // hide cursor

    int row, col;
    getmaxyx(stdscr, row, col);
    nodelay(stdscr, TRUE);                                             // enable non-blocking getch()

    while(1){
        clear();                                                       // clear screen
        time_t current_time = time(NULL);                              // get time
        struct tm* local_time = localtime(&current_time);

        char time_str[9];                                              // format
        strftime(time_str, sizeof(time_str), "%H:%M:%S", local_time);
        int time_len = strlen(time_str);                               // compute screen center pos
        int x = (col - time_len) / 2;
        int y = row / 2;

        mvprintw(y, x, "%s", time_str);                                // print time on screen
        refresh();                                                     // screen refresh
        napms(1000);                                                   // wait 1s

        int ch = getch();                                              // get user input
        if(ch != ERR){                                                 // if any key is pressed
            break;                                                     // exit loop
        }
    }
    endwin();                                                          // cleanup & exit
    return 0;
}

int main(){
    int fd = open("/dev/pts/1", O_RDWR);                               // O_RDWR, O_WRONLY, O_RDONLY
    if(fd == -1){
        perror("Failed to open /dev/tty2");
        return 1;
    }
    dup2(fd, STDOUT_FILENO);                                           // redirect stdout to tty
    dup2(fd, STDIN_FILENO);                                            // redirect stdin from tty
    close(fd);

    timeclock();                                                       // startup text clock app
    return 0;
}
```
todo: ask a stackoverflow question about: can this program detect keystroke over tty, when displaying on
another terminal.

<hr>

### # a feature-rich tui timeclock (ncurses)
make a ncurses app as an screen saver, if user is inactive in certain timeout for a ssh connection.

1 ttyclock.h: contains struct ttyclock data representation & prototype definition.

```text
#ifndef TTYCLOCK_H_INCLUDED
#define TTYCLOCK_H_INCLUDED

#include <assert.h>
#include <errno.h>
#include <getopt.h>
#include <signal.h>
#include <stdbool.h>
#include <stdlib.h>
#include <string.h>
#include <sys/select.h>
#include <sys/stat.h>
#include <sys/types.h>
#include <locale.h>
#include <time.h>
#include <unistd.h>
#include <ncurses.h>

#define NORMFRAMEW 35
#define SECFRAMEW  54
#define DATEWINH   3
#define AMSIGN     " [AM]"
#define PMSIGN     " [PM]"

typedef struct {                       // ttyclock struct
    bool running;                      // running or not
    SCREEN* ttyscr;                    // terminal variables
    char* tty;
    int bg;

    struct {                           // running option
        bool second;
        bool screensaver;
        bool twelve;
        bool center;
        bool rebound;
        bool date;
        bool utc;
        bool box;
        bool noquit;
        char format[100];
        int  color;
        bool bold;
        long delay;
        bool blink;
        long nsdelay;
    } option;

    struct {                                             // clock geometry
         int x, y, w, h;
         int a, b;                                       // for clock rebound usage
    } geo;

    struct {                                             // date content
         unsigned int hour[2];
         unsigned int minute[2];
         unsigned int second[2];
         char datestr[256];
         char old_datestr[256];
    } date;

    struct tm* tm;                                       // time.h utils
    time_t lt;

    char* meridiem;                                      // clock member
    WINDOW* framewin;
    WINDOW* datewin;

} ttyclock_t;

void init(void);                                         // prototypes
void signal_handler(int signal);
void update_hour(void);
void draw_number(int n, int x, int y);
void draw_clock(void);
void clock_move(int x, int y, int w, int h);
void set_second(void);
void set_center(bool b);
void set_box(bool b);
void key_event(void);

ttyclock_t ttyclock;                                     // global variable

const bool number[][15] = {                              // number matrix
    {1,1,1,1,0,1,1,0,1,1,0,1,1,1,1},                     // 0
    {0,0,1,0,0,1,0,0,1,0,0,1,0,0,1},                     // 1
    {1,1,1,0,0,1,1,1,1,1,0,0,1,1,1},                     // 2
    {1,1,1,0,0,1,1,1,1,0,0,1,1,1,1},                     // 3
    {1,0,1,1,0,1,1,1,1,0,0,1,0,0,1},                     // 4
    {1,1,1,1,0,0,1,1,1,0,0,1,1,1,1},                     // 5
    {1,1,1,1,0,0,1,1,1,1,0,1,1,1,1},                     // 6
    {1,1,1,0,0,1,0,0,1,0,0,1,0,0,1},                     // 7
    {1,1,1,1,0,1,1,1,1,1,0,1,1,1,1},                     // 8
    {1,1,1,1,0,1,1,1,1,0,0,1,1,1,1},                     // 9
};

#endif                                                   // TTYCLOCK_H_INCLUDED
```

2 clock.h: contains clock ncurses app startup entrance, tui management logic\...

```text
#include "ttyclock.h"

void init(void){
    struct sigaction sig;
    setlocale(LC_TIME,"");

    ttyclock.bg = COLOR_BLACK;                           // init ncurses
    if(ttyclock.tty){
        FILE *ftty = fopen(ttyclock.tty, "r+");
        if(!ftty){
            fprintf(stderr, "tty-clock: error: '%s' couldn't be opened: %s.\n",
                    ttyclock.tty, strerror(errno));
            exit(EXIT_FAILURE);
        }
        ttyclock.ttyscr = newterm(NULL, ftty, ftty);
        assert(ttyclock.ttyscr != NULL);
        set_term(ttyclock.ttyscr);
    }
    else{
        initscr();
    }

    cbreak();
    noecho();
    keypad(stdscr, true);
    start_color();
    curs_set(false);
    clear();

    if(use_default_colors() == OK){                      // init default terminal color
        ttyclock.bg = -1;
    }

    init_pair(0, ttyclock.bg, ttyclock.bg);              // init color pair
    init_pair(1, ttyclock.bg, ttyclock.option.color);
    init_pair(2, ttyclock.option.color, ttyclock.bg);
    refresh();

    sig.sa_handler = signal_handler;                     // init signal handler
    sig.sa_flags = 0;
    sigaction(SIGTERM,  &sig, NULL);
    sigaction(SIGINT,   &sig, NULL);
    sigaction(SIGSEGV,  &sig, NULL);

    ttyclock.running = true;                             // init global struct
    if(!ttyclock.geo.x){
        ttyclock.geo.x = 0;
    }
    if(!ttyclock.geo.y){
        ttyclock.geo.y = 0;
    }
    if(!ttyclock.geo.a){
        ttyclock.geo.a = 1;
    }
    if(!ttyclock.geo.b){
        ttyclock.geo.b = 1;
    }
    ttyclock.geo.w = (ttyclock.option.second) ? SECFRAMEW : NORMFRAMEW;
    ttyclock.geo.h = 7;
    ttyclock.tm = localtime(&(ttyclock.lt));
    if(ttyclock.option.utc){
        ttyclock.tm = gmtime(&(ttyclock.lt));
    }
    ttyclock.lt = time(NULL);
    update_hour();

    ttyclock.framewin = newwin(                          // create clock window
        ttyclock.geo.h,
        ttyclock.geo.w,
        ttyclock.geo.x,
        ttyclock.geo.y
    );
    if(ttyclock.option.box){
        box(ttyclock.framewin, 0, 0);
    }
    if(ttyclock.option.bold){
        wattron(ttyclock.framewin, A_BLINK);
    }

    ttyclock.datewin = newwin(                           // create date window
        DATEWINH, strlen(ttyclock.date.datestr) + 2,
        ttyclock.geo.x + ttyclock.geo.h - 1,
        ttyclock.geo.y + (ttyclock.geo.w / 2) - (strlen(ttyclock.date.datestr) / 2) - 1
    );
    if(ttyclock.option.box && ttyclock.option.date){
        box(ttyclock.datewin, 0, 0);
    }
    clearok(ttyclock.datewin, true);
    set_center(ttyclock.option.center);
    nodelay(stdscr, true);

    if(ttyclock.option.date){
        wrefresh(ttyclock.datewin);
    }
    wrefresh(ttyclock.framewin);
    return;
}

void signal_handler(int signal){
    switch(signal){
    case SIGINT:
    case SIGTERM:
        ttyclock.running = false;
        break;
    case SIGSEGV:
        endwin();
        fprintf(stderr, "Segmentation fault.\n");
        exit(EXIT_FAILURE);
        break;
    }
    return;
}

void cleanup(void){
    if(ttyclock.ttyscr){
        delscreen(ttyclock.ttyscr);
    }
    free(ttyclock.tty);
}

void update_hour(void){
    int ihour;
    char tmpstr[128];

    ttyclock.lt = time(NULL);
    ttyclock.tm = localtime(&(ttyclock.lt));
    if(ttyclock.option.utc){
        ttyclock.tm = gmtime(&(ttyclock.lt));
    }

    ihour = ttyclock.tm->tm_hour;

    if(ttyclock.option.twelve){                                               // manage hour for 12 mod
        ttyclock.meridiem = ((ihour >= 12) ? PMSIGN : AMSIGN);
    }
    else{
        ttyclock.meridiem = "\0";
    }
    ihour = ((ttyclock.option.twelve && ihour > 12) ? (ihour - 12) : ihour);
    ihour = ((ttyclock.option.twelve && !ihour) ? 12 : ihour);

    ttyclock.date.hour[0] = ihour / 10;                                       // set hour
    ttyclock.date.hour[1] = ihour % 10;

    ttyclock.date.minute[0] = ttyclock.tm->tm_min / 10;                       // set minutes
    ttyclock.date.minute[1] = ttyclock.tm->tm_min % 10;

    strcpy(ttyclock.date.old_datestr, ttyclock.date.datestr);                 // set date string
    strftime(tmpstr, sizeof(tmpstr), ttyclock.option.format, ttyclock.tm);
    sprintf(ttyclock.date.datestr, "%s%s", tmpstr, ttyclock.meridiem);

    ttyclock.date.second[0] = ttyclock.tm->tm_sec / 10;                       // set seconds
    ttyclock.date.second[1] = ttyclock.tm->tm_sec % 10;

    return;
}

void draw_number(int n, int x, int y){
    int i, sy = y;
    for(i=0; i<30; ++i, ++sy){
        if(sy == y+6){
            sy = y;
            ++x;
        }
        if(ttyclock.option.bold){
            wattron(ttyclock.framewin, A_BLINK);
        }
        else{
            wattroff(ttyclock.framewin, A_BLINK);
        }
        wbkgdset(ttyclock.framewin, COLOR_PAIR(number[n][i/2]));
        mvwaddch(ttyclock.framewin, x, sy, ' ');
    }
    wrefresh(ttyclock.framewin);
    return;
}

void draw_clock(void){
    if(ttyclock.option.date && !ttyclock.option.rebound &&
       strcmp(ttyclock.date.datestr, ttyclock.date.old_datestr) != 0){
        clock_move(ttyclock.geo.x, ttyclock.geo.y, ttyclock.geo.w, ttyclock.geo.h);
    }

    draw_number(ttyclock.date.hour[0], 1, 1);                 // draw hour numbers
    draw_number(ttyclock.date.hour[1], 1, 8);
    chtype dotcolor = COLOR_PAIR(1);
    if(ttyclock.option.blink && time(NULL) % 2 == 0){
        dotcolor = COLOR_PAIR(2);
    }

    wbkgdset(ttyclock.framewin, dotcolor);                    // 2 dot for number separation
    mvwaddstr(ttyclock.framewin, 2, 16, "  ");
    mvwaddstr(ttyclock.framewin, 4, 16, "  ");
    draw_number(ttyclock.date.minute[0], 1, 20);              // draw minute numbers
    draw_number(ttyclock.date.minute[1], 1, 27);

    if(ttyclock.option.bold){                                 // draw the date
        wattron(ttyclock.datewin, A_BOLD);
    }
    else{
        wattroff(ttyclock.datewin, A_BOLD);
    }

    if(ttyclock.option.date){
        wbkgdset(ttyclock.datewin, (COLOR_PAIR(2)));
        mvwprintw(ttyclock.datewin, (DATEWINH / 2), 1, "%s", ttyclock.date.datestr);
        wrefresh(ttyclock.datewin);
    }

    if(ttyclock.option.second){                               // draw second if option -s enabled
        wbkgdset(ttyclock.framewin, dotcolor);                // 2 dot for number separation
        mvwaddstr(ttyclock.framewin, 2, NORMFRAMEW, "  ");
        mvwaddstr(ttyclock.framewin, 4, NORMFRAMEW, "  ");
        draw_number(ttyclock.date.second[0], 1, 39);          // draw second numbers
        draw_number(ttyclock.date.second[1], 1, 46);
    }

    return;
}

void clock_move(int x, int y, int w, int h){
    wbkgdset(ttyclock.framewin, COLOR_PAIR(0));                              // erase border for clean move
    wborder(ttyclock.framewin, ' ', ' ', ' ', ' ', ' ', ' ', ' ', ' ');
    werase(ttyclock.framewin);
    wrefresh(ttyclock.framewin);

    if(ttyclock.option.date){
        wbkgdset(ttyclock.datewin, COLOR_PAIR(0));
        wborder(ttyclock.datewin, ' ', ' ', ' ', ' ', ' ', ' ', ' ', ' ');
        werase(ttyclock.datewin);
        wrefresh(ttyclock.datewin);
    }

    mvwin(ttyclock.framewin, (ttyclock.geo.x = x), (ttyclock.geo.y = y));    // frame window movement
    wresize(ttyclock.framewin, (ttyclock.geo.h = h), (ttyclock.geo.w = w));

    if(ttyclock.option.date){                                                // date window movement
        mvwin(ttyclock.datewin, ttyclock.geo.x + ttyclock.geo.h - 1,
              ttyclock.geo.y + (ttyclock.geo.w / 2) - (strlen(ttyclock.date.datestr) / 2) - 1);
        wresize(ttyclock.datewin, DATEWINH, strlen(ttyclock.date.datestr) + 2);
        if(ttyclock.option.box){
            box(ttyclock.datewin,  0, 0);
        }
    }
    if(ttyclock.option.box){
        box(ttyclock.framewin, 0, 0);
    }

    wrefresh(ttyclock.framewin);
    wrefresh(ttyclock.datewin);
    return;
}

void clock_rebound(void){                                       // useless but fun
    if(!ttyclock.option.rebound){
        return;
    }
    if(ttyclock.geo.x < 1){
        ttyclock.geo.a = 1;
    }
    if(ttyclock.geo.x > (LINES - ttyclock.geo.h - DATEWINH)){
        ttyclock.geo.a = -1;
    }
    if(ttyclock.geo.y < 1){
        ttyclock.geo.b = 1;
    }
    if(ttyclock.geo.y > (COLS - ttyclock.geo.w - 1)){
        ttyclock.geo.b = -1;
    }
    clock_move(ttyclock.geo.x + ttyclock.geo.a, ttyclock.geo.y + ttyclock.geo.b, ttyclock.geo.w, ttyclock.geo.h);
    return;
}

void set_second(void){
    int new_w = (((ttyclock.option.second = !ttyclock.option.second)) ? SECFRAMEW : NORMFRAMEW);
    int y_adj;
    for(y_adj = 0; (ttyclock.geo.y - y_adj) > (COLS - new_w - 1); ++y_adj);
    clock_move(ttyclock.geo.x, (ttyclock.geo.y - y_adj), new_w, ttyclock.geo.h);
    set_center(ttyclock.option.center);
    return;
}

void set_center(bool b){
    if((ttyclock.option.center = b)){
        ttyclock.option.rebound = false;
        clock_move(
            (LINES/2 - (ttyclock.geo.h/2)), (COLS/2-(ttyclock.geo.w/2)), ttyclock.geo.w, ttyclock.geo.h
        );
    }
    return;
}

void set_box(bool b){
    ttyclock.option.box = b;

    wbkgdset(ttyclock.framewin, COLOR_PAIR(0));
    wbkgdset(ttyclock.datewin, COLOR_PAIR(0));

    if(ttyclock.option.box){
        wbkgdset(ttyclock.framewin, COLOR_PAIR(0));
        wbkgdset(ttyclock.datewin, COLOR_PAIR(0));
        box(ttyclock.framewin, 0, 0);
        box(ttyclock.datewin,  0, 0);
    }
    else{
        wborder(ttyclock.framewin, ' ', ' ', ' ', ' ', ' ', ' ', ' ', ' ');
        wborder(ttyclock.datewin, ' ', ' ', ' ', ' ', ' ', ' ', ' ', ' ');
    }

    wrefresh(ttyclock.datewin);
    wrefresh(ttyclock.framewin);
}

void key_event(void){
    int i, c;

    struct timespec length = { ttyclock.option.delay, ttyclock.option.nsdelay };

    fd_set rfds;
    FD_ZERO(&rfds);
    FD_SET(STDIN_FILENO, &rfds);

    if(ttyclock.option.screensaver){
        c = wgetch(stdscr);
        if(c != ERR && ttyclock.option.noquit == false){
            ttyclock.running = false;
        }
        else{
            nanosleep(&length, NULL);
            for(i = 0; i < 8; ++i){
                if(c == (i + '0')){
                    ttyclock.option.color = i;
                    init_pair(1, ttyclock.bg, i);
                    init_pair(2, i, ttyclock.bg);
                }
            }
        }
        return;
    }
    switch(c = wgetch(stdscr)){
        case KEY_RESIZE:
            endwin();
            init();
            break;
        case KEY_UP:
        case 'k':
        case 'K':
            if(ttyclock.geo.x >= 1 && !ttyclock.option.center){
                clock_move(ttyclock.geo.x - 1, ttyclock.geo.y, ttyclock.geo.w, ttyclock.geo.h);
            }
            break;
        case KEY_DOWN:
        case 'j':
        case 'J':
            if(ttyclock.geo.x <= (LINES - ttyclock.geo.h - DATEWINH) && !ttyclock.option.center){
                clock_move(ttyclock.geo.x + 1, ttyclock.geo.y, ttyclock.geo.w, ttyclock.geo.h);
            }
            break;
        case KEY_LEFT:
        case 'h':
        case 'H':
            if(ttyclock.geo.y >= 1 && !ttyclock.option.center){
                clock_move(ttyclock.geo.x, ttyclock.geo.y - 1, ttyclock.geo.w, ttyclock.geo.h);
            }
            break;
        case KEY_RIGHT:
        case 'l':
        case 'L':
            if(ttyclock.geo.y <= (COLS - ttyclock.geo.w - 1) && !ttyclock.option.center){
                clock_move(ttyclock.geo.x, ttyclock.geo.y + 1, ttyclock.geo.w, ttyclock.geo.h);
            }
            break;
        case 'q':
        case 'Q':
            if(ttyclock.option.noquit == false){
                ttyclock.running = false;
            }
            break;
        case 's':
        case 'S':
            set_second();
            break;
        case 't':
        case 'T':
            ttyclock.option.twelve = !ttyclock.option.twelve;
            update_hour();  // set new ttyclock.date.datestr & win resize 
            clock_move(ttyclock.geo.x, ttyclock.geo.y, ttyclock.geo.w, ttyclock.geo.h);
            break;
        case 'c':
        case 'C':
            set_center(!ttyclock.option.center);
            break;
        case 'b':
        case 'B':
            ttyclock.option.bold = !ttyclock.option.bold;
            break;
        case 'r':
        case 'R':
            ttyclock.option.rebound = !ttyclock.option.rebound;
            if(ttyclock.option.rebound && ttyclock.option.center){
                ttyclock.option.center = false;
            }
            break;
        case 'x':
        case 'X':
            set_box(!ttyclock.option.box);
            break;
        case '0': case '1': case '2': case '3':
        case '4': case '5': case '6': case '7':
            i = c - '0';
            ttyclock.option.color = i;
            init_pair(1, ttyclock.bg, i);
            init_pair(2, i, ttyclock.bg);
            break;
        default:
            pselect(1, &rfds, NULL, NULL, &length, NULL);
        }
        return;
}

int tclock(int argc, char **argv){
    int c;

    memset(&ttyclock, 0, sizeof(ttyclock_t));                                // alloc ttyclock

    ttyclock.option.date = true;

    strncpy(ttyclock.option.format, "%F", sizeof (ttyclock.option.format));  // default date format
    ttyclock.option.color = COLOR_GREEN;                                     // default color2
    ttyclock.option.delay = 1;                                               // default delay: 1fps
    ttyclock.option.nsdelay = 0;
    ttyclock.option.blink = false;

    atexit(cleanup);

    while((c = getopt(argc, argv, "iuvsScbtrhBxnDC:f:d:T:a:")) != -1){
        switch(c){
        case 'h':
        default:
            printf("usage : tty-clock [-iuvsScbtrahDBxn] [-C [0-7]] [-f format] [-d delay] [-a nsdelay] [-T tty] \n"
                   "    -s            Show seconds                                   \n"
                   "    -S            Screensaver mode                               \n"
                   "    -x            Show box                                       \n"
                   "    -c            Set the clock at the center of the terminal    \n"
                   "    -C [0-7]      Set the clock color                            \n"
                   "    -b            Use bold colors                                \n"
                   "    -t            Set the hour in 12h format                     \n"
                   "    -u            Use UTC time                                   \n"
                   "    -T tty        Display the clock on the specified terminal    \n"
                   "    -r            Do rebound the clock                           \n"
                   "    -f format     Set the date format                            \n"
                   "    -n            Don't quit on keypress                         \n"
                   "    -v            Show tty-clock version                         \n"
                   "    -i            Show some info about tty-clock                 \n"
                   "    -h            Show this page                                 \n"
                   "    -D            Hide date                                      \n"
                   "    -B            Enable blinking colon                          \n"
                   "    -d delay      Set the delay between two redraws of the clock. Default 1s. \n"
                   "    -a nsdelay    Additional delay between two redraws in nanoseconds. Default 0ns.\n");
            exit(EXIT_SUCCESS);
            break;
        case 'i':
            puts("TTY-Clock 2 © by Martin Duquesnoy (xorg62@gmail.com), Grey (grey@greytheory.net)");
            exit(EXIT_SUCCESS);
            break;
        case 'u':
            ttyclock.option.utc = true;
            break;
        case 'v':
            puts("TTY-Clock 2 © devel version");
            exit(EXIT_SUCCESS);
            break;
        case 's':
            ttyclock.option.second = true;
            break;
        case 'S':
            ttyclock.option.screensaver = true;
            break;
        case 'c':
            ttyclock.option.center = true;
            break;
        case 'b':
            ttyclock.option.bold = true;
            break;
        case 'C':
            if(atoi(optarg) >= 0 && atoi(optarg) < 8){
                ttyclock.option.color = atoi(optarg);
            }
            break;
        case 't':
            ttyclock.option.twelve = true;
            break;
        case 'r':
            ttyclock.option.rebound = true;
            break;
        case 'f':
            strncpy(ttyclock.option.format, optarg, 100);
            break;
        case 'd':
            if(atol(optarg) >= 0 && atol(optarg) < 100){
                ttyclock.option.delay = atol(optarg);
            }
            break;
        case 'D':
            ttyclock.option.date = false;
            break;
        case 'B':
            ttyclock.option.blink = true;
            break;
        case 'a':
            if(atol(optarg) >= 0 && atol(optarg) < 1000000000){
                ttyclock.option.nsdelay = atol(optarg);
            }
            break;
        case 'x':
            ttyclock.option.box = true;
            break;
        case 'T':
            struct stat sbuf;
            if(stat(optarg, &sbuf) == -1){
                fprintf(stderr, "tty-clock: error: couldn't stat '%s': %s.\n", optarg, strerror(errno));
                exit(EXIT_FAILURE);
            }
            else if(!S_ISCHR(sbuf.st_mode)){
                fprintf(stderr, "tty-clock: error: '%s' doesn't appear to be a character device.\n", optarg);
                exit(EXIT_FAILURE);
            }
            else{
                free(ttyclock.tty);
                ttyclock.tty = strdup(optarg);
            }
            break;
        case 'n':
            ttyclock.option.noquit = true;
            break;
        }
    }

    init();
    attron(A_BLINK);
    while(ttyclock.running){
        clock_rebound();
        update_hour();
        draw_clock();
        key_event();
    }
    endwin();

    return 0;
}
```

3 test.c: main file of tui clock app based on ncurse lib.

```text
// gcc -o out test.c -lncurses          (for macos)
// gcc -o out test.c -lncurses -ltinfo  (for linux)

#include <stdio.h>
#include <unistd.h>
#include <fcntl.h>

#include "clock.h"

int main(int argc, char** argv){
    int fd = open("/dev/ttys000", O_RDWR);      // fd to represent a tty (O_RDWR, O_WRONLY, O_RDONLY)
    // int fd = open("/dev/ttys002", O_RDWR);)
    if(fd == -1){
        perror("failed to open tty device!");
        return 1;
    }
    dup2(fd, STDOUT_FILENO);                    // redirect stdout to tty
    dup2(fd, STDIN_FILENO);                     // redirect stdin from tty (optional)
    close(fd);

    tclock(argc, argv);                         // startup app

    return 0;
}
```

test the program with the fd reference current working tty (/dev/ttys000), then everything work out fine.

however, if set the test.c program to direct the clock app to another terminal (/dev/ttys002),
rather than current, some issue will occur: the terminal display is ok for the first time,
but wont response to any user-input.

try modify the test.c a bit: comment out the line contains 'dup2(fd, STDIN_FILENO)', set the fd
represent another terminal, then the clock display fine at that terminal for the first time,
and typing inside the working tty, the clock app on another tty got controled.

todo: find the why the clock app wont listen to the user input from another tty.

<hr>

### # todo: integrate the above program with crontab jobs for timer checkup
