---
layout: post
title: "go defer function (golang)"
author: "melon"
date: 2023-10-09 20:57
categories: "2023"
tags:
  - golang
  - ongoing
---

### # simple progrom to illustrate the trick defer function
```text
package main

import (
    "fmt"
    "strings"
    "net/http"
    "golang.org/x/net/html"                                     // non official pkg
)

func forEachNode(n *html.Node, pre, post func(n *html.Node)) {
    if pre != nil {
        pre(n)                                                  // call pre()
    }
    for c := n.FirstChild; c != nil; c = c.NextSibling {        // recursively traverse child nodes
        forEachNode(c, pre, post)
    }
    if post != nil {                                            // call post()
        post(n)
    }
}

func title(url string) error {                                  // funciton without defer trick
    resp, err := http.Get(url)
    if err != nil {
        return err
    }
    ct := resp.Header.Get("Content-Type")                                          // get content-type
    if ct != "text/html" && !strings.HasPrefix(ct,"text/html;") {                  // check "text/html;charset=utf-8"
        resp.Body.Close()                                                          // close
        return fmt.Errorf("%s has type %s, not text/html",url, ct)
    }
    doc, err := html.Parse(resp.Body)                                              // parse
    resp.Body.Close()                                                              // close
    if err != nil {
        return fmt.Errorf("parsing %s as HTML: %v", url,err)
    }
    visitNode := func(n *html.Node) {                                              // get doc title
        if n.Type == html.ElementNode && n.Data == "title"&&n.FirstChild != nil {
            fmt.Println(n.FirstChild.Data)
        }
    }
    forEachNode(doc, visitNode, nil)                                               // show each
    return nil
}

func newtitle(url string) error {                                                  // function with defer trick
    resp, err := http.Get(url)
    if err != nil {
        return err
    }
    defer resp.Body.Close()                                                        // can cover all close situations
    ct := resp.Header.Get("Content-Type")
    if ct != "text/html" && !strings.HasPrefix(ct,"text/html;") {
        return fmt.Errorf("%s has type %s, not text/html",url, ct)
    }
    doc, err := html.Parse(resp.Body)
    if err != nil {
        return fmt.Errorf("parsing %s as HTML: %v", url,err)
    }
    visitNode := func(n *html.Node) {                                              // get doc title
        if n.Type == html.ElementNode && n.Data == "title"&&n.FirstChild != nil {
            fmt.Println(n.FirstChild.Data)
        }
    }
    forEachNode(doc, visitNode, nil)
    return nil
}

func main() {
    urls := []string{                             // arr of string url
        "https://github.com", 
        "https://www.baidu.com",
    }

    for _, url := range urls {                    // print title(including subnode) for each url
        // err := title(url)
        err := newtitle(url)
        if err != nil {
            fmt.Println(err)
            continue
        }
        fmt.Println("Title retrieved for:", url)
    }
}
```

<hr>

### # build & run the program
The method to compile & run the above code with external pkg "golang.org/x/net" is a slightly different compare to normal single go script.

Firstly, try to build the go program directly as normal:
```text
$ go build defer.go
defer.go:7:5: no required module provides package golang.org/x/net/html: go.mod file not found in current directory or any parent directory; see 'go help modules'
```
The error seems to be the imported pkg not found, so download it and build again:
```text
$ go get "golang.org/x/net/html"
$ go build defer.go
defer.go:7:5: no required module provides package golang.org/x/net/html: go.mod file not found in current directory or any parent directory; see 'go help modules'
```
So far, even with the pkg installed in $GOPATH, we still cannot make it work in single build cmd.  
Thus, we try the followings to setup current folder as a module:
```text
$ go mod init melon/test
go: creating new go.mod: module melon/test
go: to add module requirements and sums:
        go mod tidy
```
The golang.org/x/net is already downloaded inside $GOPATH/pkg/mod/, try build again, but still failed:
```text
$ go build defer.go
defer.go:7:5: no required module provides package golang.org/x/net/html; to add it:
        go get golang.org/x/net/html
```
try sync the external module by:
```text
$ go mod tidy
go: finding module for package golang.org/x/net/html
go: found golang.org/x/net/html in golang.org/x/net v0.16.0
```
try build it again and run:
```text
$ go build defer.go
$ ./defer
GitHub: Let’s build from here · GitHub
Python
JavaScript
Go
Title retrieved for: https://github.com
Get "https://www.baidu.com": dial tcp 103.235.46.40:443: i/o timeout
```
