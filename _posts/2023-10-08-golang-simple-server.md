---
layout: post
title: "go server evolution (golang)"
author: "melon"
date: 2023-10-08 22:19
categories: "2023"
tags:
  - golang
  - ongoing
---

### # a simple echo server
```text
package main                                                   // a simple echo server

import (
    "fmt"
    "log"
    "net/http"
)

func main() {
    http.HandleFunc("/", handler)                              // each request calls handler
    // log.Fatal(http.ListenAndServe("localhost:18000", nil))  // only listening http request from server local
    log.Fatal(http.ListenAndServe("0.0.0.0:18000", nil))       // listening http request from all addr 
}                                                              
                                                               
func handler(w http.ResponseWriter, r *http.Request) {         // echo path component of request url
    fmt.Fprintf(w, "URL.Path = %q\n", r.URL.Path)
}
```
we could build & test the above program by:
```text
$ go build ./server.go
```
start the above http echo server using:
```text
$ ./server
```
inside my linux cloud vm ip: 10.182.101.165, we could access the service by:  
the web page "http://10.182.101.165:18000" shows URL.Path = /  
the web page "http://10.182.101.165:18000/metung/testurl" shows URL.Path = /metung/testurl

<hr>

### # a simple echo server with global access count
```text
package main                                              // a simple echo & counter server

import (
    "fmt"
    "log"
    "net/http"
    "sync"
)

var mu sync.Mutex
var count int                                             // global count

func main() {                                             // each time a request come, a new goroutine is created
    http.HandleFunc("/", handler)                         // regist echo path handler, echo path & add count
    http.HandleFunc("/count", counter)                    // regist echo count handler, show times of matched access
    log.Fatal(http.ListenAndServe("0.0.0.0:18000", nil))
}

func handler(w http.ResponseWriter, r *http.Request) {    // echo service handler
    mu.Lock()                                             // lock the critical section: count
    count++                                               // in high concurrency scenario, avoid multiple routine racing
    mu.Unlock()                                           // unlock
    fmt.Fprintf(w, "URL.Path = %q\n", r.URL.Path)
}

func counter(w http.ResponseWriter, r *http.Request) {    // count record & echo service handler
    mu.Lock()                                             // lock
    fmt.Fprintf(w, "Count %d\n", count)
    mu.Unlock()                                           // unlock
}
```
start the above http echo server using:
```text
$ ./server
```
test the functionality of the above server by:
```text
open the web page "http://10.182.101.165:18000/count" shows: Count 0,
refresh the web page "http://10.182.101.165:18000/count" by 3 times, finally: Count 3 
in another tab of safari, access "http://10.182.101.165:18000/requesturl",
refresh the web page "http://10.182.101.165:18000/count" to check the result, shows: Count 6
```
why the count is added by 3 rathen than 2?
```text
the access of "http://10.182.101.165:18000/requesturl" add 1 count, 
the access of "http://10.182.101.165:18000/count" add 1 count, 
but actually the count is added by 3.

Checking the net/http url matching rules, when accessing the site, will firstly access /favicon.ico, 
which hit the rules of first handler '/' -> add count by 1.
```
So, the rectified solution to the above problem is: lanunch the "echo & add 1" service at "/echo" not simply "/", whcih will not cause the initial access to /favicon.icon to be counted:
```text
package main                                              // a simple echo & counter server

import (
    "fmt"
    "log"
    "net/http"
    "sync"
)

var mu sync.Mutex
var count int                                             // global count

                                                          // each time a request come, a new goroutine is created 
func main() {                                             // for exec the corresponding handler
    http.HandleFunc("/echo", handler)                     // set echo service using url start with "echo"
    http.HandleFunc("/count", counter)
    // http.HandleFunc("/echo/", handler)                 // try 3
    // http.HandleFunc("/echo/count/", counter)
    log.Fatal(http.ListenAndServe("0.0.0.0:18000", nil))
}

func handler(w http.ResponseWriter, r *http.Request) {    // echo service handler
    mu.Lock()                                             // lock the critical section: count
    count++                                               // in high concurrency scenario, avoid multiple routine racing
    mu.Unlock()                                           // unlock
    fmt.Fprintf(w, "URL.Path = %q\n", r.URL.Path)
}

func counter(w http.ResponseWriter, r *http.Request) {    // count record & echo service handler
    mu.Lock()                                             // lock
    fmt.Fprintf(w, "Count %d\n", count)
    mu.Unlock()                                           // unlock
}
```
the above code only add count when accessing the "/echo" url, but will not add count when accessing "/count" anymore.  
the solution is ready and to be summarized here.

<hr>

### # the right way to design the url layered as service API: RESTful API?
