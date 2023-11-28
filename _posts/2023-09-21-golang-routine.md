---
layout: post
title: "go routine (golang)"
author: "melon"
date: 2023-09-21 21:15
categories: "2023"
tags:
  - golang
  - ongoing
---

### # goroutine introduction
when goroutine send/recv to channel, the routine is blocked on that channel till next routine send/recv to the channel.

The order of goroutine exec can not be guaranteed, which is scheduled by go runtime schedualor with a minimal overhead of context switching, we could setup the priority of goroutine, but there's strong reason of not doing that.

<hr>

### # goroutine sample code
the following program fetches urls in parallel and reports their times and sizes.
```text
package main

import (
    "fmt"
    "io"
    "io/ioutil"
    "net/http"
    "os"
    "time"
)

func main() {                                                   // main routine -> program entrance
    start := time.Now()
    ch := make(chan string)
    for _, url := range os.Args[1:] {
        go fetch(url, ch)                                       // start goroutine
    }
    for range os.Args[1:] {
        fmt.Println(<-ch)                                       // receive from ch
    }
    fmt.Printf("%.2fs elapsed\n", time.Since(start).Seconds())
}

func fetch(url string, ch chan<- string) {
    start := time.Now()
    resp, err := http.Get(url)                                  // http.GET -> resp
    if err != nil {                                             // err
        ch <- fmt.Sprint(err)
        return
    }
    nbytes, err := io.Copy(ioutil.Discard, resp.Body)           // just return the num of byte not the content
    resp.Body.Close()                                           // don't leak resources
    if err != nil {                                             // err
        ch <- fmt.Sprintf("while reading %s: %v", url, err)
        return
    }
    secs := time.Since(start).Seconds()
    ch <- fmt.Sprintf("%.2fs  %7d  %s", secs, nbytes, url)      // send res into ch
}
```
we could build & test the above program by:
```text
$ go build ./routine.go
```
```text
$ ./routine "https://golang.org" "http://gopl.io" "https://godoc.org"
1.94s    31630  https://godoc.org
2.17s     4154  http://gopl.io
Get "https://golang.org": dial tcp 142.250.71.81:443: i/o timeout
30.03s elapsed
```

ref: https://gopl-zh.github.io/ch1/ch1-06.html
