---
layout: post
title: "no std::endl abuse (c++)"
author: "melon"
date: 2023-10-15 12:46
categories: "2023"
tags:
  - cpp
---

it is a common practice to use std::endl for every newlines print:

```text
std::cout << "hello world!" << std::endl;
```

for programs with very little i/o operations this practice is acceptable,
but with the bulk of i/o operations increasing, the efficiency of the program is degraded.

a equivalent form of the line above:

```text
std::cout << "hello world!\n" << std::flush;
```

thus, std::endl not only add newline on print, but also flush the buffer each time called.

<hr>

### # why avoid std::endl each time write to stream
1 each time the buffer is flushed, a request has to be made to the os and the requests
to kernel are really expensive.

2 it is un-reasonable to flush the buffer every time write to stream,
since the buffers get flushed automatically when they get full.

3 even in the rare cases there are actual need to perform flushing,
we can explicitly specify the operation using std::flush.

<hr>

### # performance impact: \n vs std::endl;
```text
#include <iostream>
#include <chrono>                                  // for high resolution duration record
#include <fstream>

using namespace std;
using namespace std::chrono;

int main(){
    ofstream file1("file1.txt");
    ofstream file2("file2.txt");

    auto start = high_resolution_clock::now();     // write to file1 with each line follows a std::endl;
    for(int i=0; i<100000; i++){
        file1 << "hello world" << std::endl;
    }
    auto stop = high_resolution_clock::now();
    auto duration = duration_cast<microseconds>(stop-start);
    cout << "writing to file using endl took "
         << duration.count() << " microseconds" << std::endl;

    start = high_resolution_clock::now();          // write to file2 with each line end with \n
    for(int i=0; i<100000; i++){
        file2 << "hello world\n";
    }
    stop = high_resolution_clock::now();
    duration = duration_cast<microseconds>(stop-start);
    cout << "writing to file using \\n took "
         << duration.count() << " microseconds"<< std::endl;

    file1.close();
    file2.close();

    return 0;
}
```

compile & execute the above program:
```text
$ g++ -g -Wall -std=c++11 -w -c t.cpp -o t.o
$ g++ -g -Wall -std=c++11 -w -o out t.o

$ ./out
writing to file using endl took 399248 microseconds
writing to file using \n took 9019 microseconds
```

the result shows: std::endl took nearly quadruple time of \n on my macos.
on some systems, the impact could be even worse, and it is guaranteed that
std::endl will take more time than printing \n.
