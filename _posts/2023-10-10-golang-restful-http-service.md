---
layout: post
title: "restful http service toy code (golang, http, restful)"
author: "melon"
date: 2023-10-10 22:38
categories: "2023"
tags:
  - golang
  - todo
---

### # golang gin http server toy code

```text
package main

import (
    // "fmt"
    "net/http"
    "github.com/gin-gonic/gin"
)

type album struct {
    ID      string   `json:"id"`
    Title   string   `json:"title"`
    Artist  string   `json:"artist"`
    Price   float64  `json:"price"`
}

var albums = []album {
    {ID: "1", Title: "Blue Train", Artist: "John Coltrane", Price: 56.99},
    {ID: "2", Title: "Jeru", Artist: "Gerry Mulligan", Price: 17.99},
    {ID: "3", Title: "Sarah Vaughan and Clifford Brown", Artist: "Sarah Vaughan", Price: 39.99},
}

func getAlbums(c *gin.Context) {
    c.IndentedJSON(http.StatusOK, albums)          // return formatted json to handler
}

func postAlbums(c *gin.Context) {
    var newAlbum album
    if err := c.BindJSON(&newAlbum); err != nil {  // bind the received json from gin.ctx to newalbum
        return
    }
    albums = append(albums, newAlbum)              // update albums
    c.IndentedJSON(http.StatusCreated, newAlbum)
}

func getAlbumbyID(c *gin.Context) {
    id := c.Param("id")                            // bind the route id from gin.ctx to id
    for _, item := range albums {
        if item.ID == id {
            c.IndentedJSON(http.StatusOK, item)
            return
        }
    }
    c.IndentedJSON(http.StatusNotFound, gin.H{"message": "album not found"})
}

func main() {
    router := gin.Default()                        // init a gin http router
    router.GET("/albums", getAlbums)               // add route for get at ip:port/albums
    router.GET("/albums/:id", getAlbumbyID)        // add route for get at ip:port/albums/id
    router.POST("/albums", postAlbums)             // add route for post at ip:port/albums
    router.Run("0.0.0.0:8080")                     // start http server listening on any addr port 8080
}
```

<hr>

### # test & debug
golang env container startup script:

```text
#!/bin/sh

proj='websrv'
image='golang:alpine3.18'
cmd='bash'

script_dir="$(realpath $(dirname "$proj"))"/"$proj"

# port mapping from inside 8080 to host 3333 (service remap)
docker run -it --rm \
    -w /"$proj" \
    -v "$script_dir":/"$proj" \
    -v /var/run/docker.sock:/var/run/docker.sock:rw \
    -p 3333:8080 \
    $image $cmd
```

init golang project deps, startup the gin.http service:

```text
$(container) go mod init melon/websrv             // name the project
$(container) go get github.com/gin-gonic/gin      // download the dep
$(container) go run main.go                       // run
```

check container listening ports, the service is fine:

```text
$(container) netstat -tuln
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State
tcp        0      0 :::8080                 :::*                    LISTEN
```

test for service handler state, encounter errors:

```text
$(container) curl -v localhost:8080/albums
* Uses proxy env variable http_proxy == 'http://10.144.1.10:8080'
*   Trying 10.144.1.10:8080...
* Connected to 10.144.1.10 (10.144.1.10) port 8080
* using HTTP/1.x
> GET http://localhost:8080/albums HTTP/1.1
> Host: localhost:8080
> User-Agent: curl/8.12.1
> Accept: */*
> Proxy-Connection: Keep-Alive
>
* Request completely sent off
< HTTP/1.1 504 Gateway Timeout
< Content-Type: text/html
< Cache-Control: no-cache
< X-XSS-Protection: 1; mode=block
< X-Content-Type-Options: nosniff
< Content-Length: 2807

<!DOCTYPE html PUBLIC "-//W3C//DTD HTML 4.01//EN">
<html>
  <head>
    <meta http-equiv="Content-Type" content="text/html; charset=UTF-8">
    <title>
      504 Gateway Timeout: remote server did not respond to the proxy
    </title>
  </head>
</html>
```

which show the http/s proxy block the connection to the service.

check the env proxy settings:

```text
$(container) env
HOSTNAME=5d27d1e730fe
DELVE_EDITOR=vim
PWD=/websrv
HOME=/root
https_proxy=http://10.144.1.10:8080               // set via docker build to enable container network access
GOLANG_VERSION=1.22.1
TERM=xterm
SHLVL=2
GOTOOLCHAIN=local
http_proxy=http://10.144.1.10:8080                // ...
PATH=/go/bin:/usr/local/go/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
CGO_ENABLED=1
GOPATH=/go
_=/usr/bin/env
```

set no proxy for localhost,127.0.0.1, that is, no use proxy server for localhost network packets:

```text
$(container) export no_proxy=localhost,127.0.0.1
$(container) env
no_proxy=localhost,127.0.0.1                      // no proxy set in env
HOSTNAME=5d27d1e730fe
DELVE_EDITOR=vim
PWD=/websrv
HOME=/root
https_proxy=http://10.144.1.10:8080
GOLANG_VERSION=1.22.1
TERM=xterm
SHLVL=2
GOTOOLCHAIN=local
http_proxy=http://10.144.1.10:8080
PATH=/go/bin:/usr/local/go/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
CGO_ENABLED=1
GOPATH=/go
_=/usr/bin/env
```

test the service inside container (same netns), the server working fine:

```text
$(container) apk add curl
$(container) curl localhost:8080/albums
[GIN] 2025/02/20 - 09:03:09 | 200 |     200.338µs |       127.0.0.1 | GET      "/albums"
[
    {
        "id": "1",
        "title": "Blue Train",
        "artist": "John Coltrane",
        "price": 56.99
    },
    {
        "id": "2",
        "title": "Jeru",
        "artist": "Gerry Mulligan",
        "price": 17.99
    },
    {
        "id": "3",
        "title": "Sarah Vaughan and Clifford Brown",
        "artist": "Sarah Vaughan",
        "price": 39.99
    }
]
```

try access the service from host machine via port mapping, which exhibit connection down (no service):

```text
$(host) curl localhost:3333/
curl: (56) Recv failure: Connection reset by peer

$(host) curl localhost:3333/albums
curl: (56) Recv failure: Connection reset by peer
```

further check the gin http server listening process, enlarge the server to listen all ip process:

```text
package main

...

func main(){
    router := gin.Default()
    ...
    // router.Run("127.0.0.1:8080")             // only listening to lo packets
    router.Run("0.0.0.0:8080")                  // reset the server listening from ${any_addr}:8080
}
```

re-test from host machine, the test result is promising:

```text
$(container) curl localhost:3333/
404 page not found

$(container) curl localhost:3333/albums
[GIN] 2025/02/20 - 09:03:09 | 200 |     200.338µs |       127.0.0.1 | GET      "/albums"
[
    {
        "id": "1",
        "title": "Blue Train",
        "artist": "John Coltrane",
        "price": 56.99
    },
    {
        "id": "2",
        "title": "Jeru",
        "artist": "Gerry Mulligan",
        "price": 17.99
    },
    {
        "id": "3",
        "title": "Sarah Vaughan and Clifford Brown",
        "artist": "Sarah Vaughan",
        "price": 39.99
    }
]
```
