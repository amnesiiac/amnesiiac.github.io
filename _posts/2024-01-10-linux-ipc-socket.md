---
layout: post
title: "inter-process communication: sockets (linux, ipc)"
author: "melon"
date: 2024-01-10 23:45
categories: "2024"
tags:
  - linux
  - ipc
---

this article covers high-end socket for ipc.

<hr>

### # types of sockets
1 unix domain sockets: enable channel-based ipc on the same physical device,
typically rely on a local file as socket addr.

2 network sockets: enable ipc over the network, with the support of tcp/udp proto.

<hr>

### # types of server/client patterns
sockets configured as streams are bidirectional, while control follows a c/s pattern:
the client init the conversation by trying to connect to a server, the server tries to
accept the connection.

1 iterative server: handle connected clients one at a time till the connection completion:
each client is handled from start to finish, then the second, and so on. the downside:
if the client handled right now hangs, all the clients waiting behind might starve to death.

2 production-grade server: support concurrent connections, typically using some mix
of multi-processing and multi-threading. e.g., the nginx server has a pool of worker processes
used for handling client requests concurrently.

<hr>

### # simple iterative server + client sample
the following sample code covers network sockets, but the server/client run on
the same machine because the server uses network address localhost (127.0.0.1) for ipc.

common utilities header (sock.h):
```text
#define PortNumber             9876
#define MaxConnects               8
#define BuffSize                256
#define ConversationLen           3
#define Host             "localhost"

void report(const char* msg, int terminate){
    perror(msg);
    if(terminate){
        exit(-1);
    }
}
```

tcp server program (server.c):
```text
#include <string.h>
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/types.h>
#include <sys/socket.h>
#include <netinet/tcp.h>
#include <arpa/inet.h>

#include "sock.h"

int main(){
    int fd = socket(                                                       // [socket]: fd
        AF_INET,                                                           // AF_INET vs AF_LOCAL
        SOCK_STREAM,                                                       // tcp: reliable, bidirectional
        0
    );
    if(fd < 0){
        report("socket", 1);
    }
        
    struct sockaddr_in saddr;                                              // buildup server's local addr
    memset(&saddr, 0, sizeof(saddr));                                      // init server addr
    saddr.sin_family = AF_INET;                                            // ipv4 AF_LOCAL
    saddr.sin_addr.s_addr = htonl(INADDR_ANY);                             // server socket is bind to any itf of host
    // saddr.sin_addr.s_addr = htonl(INADDR_LOOPBACK);                     // right! server socket is bind to lo
    // saddr.sin_addr.s_addr = htonl("127.0.0.1");                         // wrong! bind: can't assign requested addr
    saddr.sin_port = htons(PortNumber);                                    // for listening

    if(bind(fd, (struct sockaddr *) &saddr, sizeof(saddr)) < 0){           // [bind] server addr in mem
        report("bind", 1);
    }
        
    if(listen(fd, MaxConnects) < 0){                                       // [listen] for clients, up to 8 conn
        report("listen", 1);
    }

    fprintf(stderr, "listening on port %i for clients...\n", PortNumber);
    while(1){                                                              // listen forever traditionally
        struct sockaddr_in caddr;                                          // init client address
        int len = sizeof(caddr);                                           // address length could change

        int client_fd = accept(fd, (struct sockaddr*) &caddr, &len);       // [accept] block until conn established
        if(client_fd < 0){
            report("accept", 0);                                           // don't terminated, though there's a problem
            continue;
        }

        for(int i=0; i<ConversationLen; i++){
            char buffer[BuffSize + 1];                                     // init read buf
            memset(buffer, '\0', sizeof(buffer)); 
            int count = read(client_fd, buffer, sizeof(buffer));           // read
            if(count > 0){
                puts(buffer);                                              // print received content on screen
                write(client_fd, buffer, sizeof(buffer));                  // echo received msg as confirm to client
            }
        }
        close(client_fd);                                                  // close connection
    }
    return 0;
}
```

tcp client program (client.c):
```text
#include <string.h>
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/types.h>
#include <sys/socket.h>
#include <arpa/inet.h>
#include <netinet/in.h>
#include <netinet/tcp.h>
#include <netdb.h>

#include "sock.h"

const char* books[] = {
    "war and peace",
    "pride and prejudice",
    "the sound and the fury"
};

int main(){
    int sockfd = socket(                                                       // [socket] = fd
        AF_INET,                                                               // addr family: AF_INET vs AF_LOCAL
        SOCK_STREAM,                                                           // socket type: tcp
        0
    );
    if(sockfd < 0){
        report("socket", 1);
    }

    struct hostent* hptr = gethostbyname(Host);                                // lookup: localhost: 127.0.0.1
    if(!hptr){
        report("gethostbyname", 1);
    }
    if(hptr->h_addrtype != AF_INET){                                           // only support on ipv4
        report("bad address family", 1);
    }

    struct sockaddr_in saddr;                                                  // config server addr
    memset(&saddr, 0, sizeof(saddr));
    saddr.sin_family = AF_INET;
    saddr.sin_addr.s_addr = ((struct in_addr*) hptr->h_addr_list[0])->s_addr;
    saddr.sin_port = htons(PortNumber);                                        // big-endian

    if(connect(sockfd, (struct sockaddr*) &saddr, sizeof(saddr)) < 0){         // [connect]
        report("connect", 1);
    }

    puts("connect to server, about to write some stuff...");                   // write start
    for(int i=0; i<ConversationLen; i++){
        if(write(sockfd, books[i], strlen(books[i])) > 0){
            char buffer[BuffSize + 1];                                         // create read buf
            memset(buffer, '\0', sizeof(buffer));
            if(read(sockfd, buffer, sizeof(buffer)) > 0){                      // read echo
                puts(buffer);
            }
        }
    }
    puts("client done, about to exit...");                                     // write end
    close(sockfd);                                                             // close connection
    return 0;
}
```

compile & execute the server program:
```text
$ gcc -o s server.c && ./s
listening on port 9876 for clients...            # server start conn listening
war and peace                                    # puts the received msg from client
pride and prejudice
the sound and the fury
war and peace                                    # 2nd
pride and prejudice
the sound and the fury
war and peace
...
```

compile & execute the client program:
```text
$ gcc -o c client.c && ./c                       # exec client for the 1st time
connect to server, about to write some stuff...
war and peace
pride and prejudice
the sound and the fury
client done, about to exit...

$ ./c                                            # exec client for the second time
connect to server, about to write some stuff...
war and peace                                    # puts the echo back msg from server
pride and prejudice
the sound and the fury
client done, about to exit...
```

<hr>

### # unix domain sockets vs network sockets
local unix domain sockets and network sockets differ only in a few implementation details:
unix domain sockets have lower overhead and better performance. while, the basic api is
essentially the same for both.
