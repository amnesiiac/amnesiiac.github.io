---
layout: post
title: "inter-process communication: message queues (linux, ipc)"
author: "melon"
date: 2024-01-10 22:45
categories: "2024"
tags:
  - linux
  - ipc
---

pipes have to work in strict fifo manner, while msg queues can behave the same way,
but flexible enough to retrieve the bytes out of fifo order.

message queues are sequence of messages, each of which has 2 parts:  
1) the payload, which is an array of bytes (char).  
2) a type, which is int value to categorize message for flexible retrieval.

given a message queue with each message labled with an int type:
```txt
                          +-+    +-+    +-+    +-+ 
                sender--->|3|--->|2|--->|2|--->|1|--->receiver
                          +-+    +-+    +-+    +-+
```
if msg is retrieved in strict fifo manner, the receiver will get msg in 1-2-2-3
order, however, the msg queue allow other orders, e.g. the msg could be retrieved
in an order of 3-2-1-2.

<hr>

### # message queue: ipc between 2 process
common header for message definition (que.h):
```text
#define ProjectId 123
#define PathName  "que.h"             // any existing, accessible file would do
#define MsgLen    4
#define MsgCount  6

typedef struct {
    long type;                        // must be of type long
    char payload[MsgLen + 1];         // bytes in each message
} queuedMessage;
```

message sender program (sender.c):
```text
#include <stdio.h>
#include <sys/ipc.h>
#include <sys/msg.h>
#include <stdlib.h>
#include <string.h>

#include "que.h"

void report_and_exit(const char* msg){
    perror(msg);
    exit(-1);                                                             // EXIT_FAILURE
}

int main(){
    key_t key = ftok(PathName, ProjectId);                                // generate key for identify msg que
    if(key < 0){
        report_and_exit("couldn't get key...");
    }

    int qid = msgget(key, 0666 | IPC_CREAT);                              // get or create the msg que
    if(qid < 0){
        report_and_exit("couldn't get queue id...");
    }

    char* payloads[] = {"msg1", "msg2", "msg3", "msg4", "msg5", "msg6"};  // msg content
    int types[] = {1, 1, 2, 2, 3, 3};                                     // types of each msg (must be > 0)

    for(int i=0; i<MsgCount; i++){
        queuedMessage msg;                                                // build msg
        msg.type = types[i];
        strcpy(msg.payload, payloads[i]);
        msgsnd(qid, &msg, MsgLen+1, IPC_NOWAIT);                          // send msg in non-block mode
        printf("%s sent as type %i\n", msg.payload, (int) msg.type);
    }
    return 0;
}
```

message receiver program (receiver.c):
```text
#include <stdio.h> 
#include <sys/ipc.h> 
#include <sys/msg.h>
#include <stdlib.h>

#include "que.h"

void report_and_exit(const char* msg){
    perror(msg);
    exit(-1);                                                             // exit with err
}

int main(){
    key_t key = ftok(PathName, ProjectId);                                // generate key to identify a msg que
    if(key < 0){
        report_and_exit("key not gotten...");
    }

    int qid = msgget(key, 0666 | IPC_CREAT);                              // access the msg que by key if created
    if(qid < 0){
        report_and_exit("no access to queue...");
    }

    int types[] = {3, 1, 2, 1, 3, 2};                                     // msg recv priority different with sender
    for(int i=0; i<MsgCount; i++){
        queuedMessage msg;                                                // placeholder for single msg
        if(msgrcv(qid, &msg, MsgLen+1, types[i], MSG_NOERROR | IPC_NOWAIT) < 0){  // recv
            puts("msgrcv trouble...");
        }
        printf("%s received as type %i\n", msg.payload, (int) msg.type);
    }

    if(msgctl(qid, IPC_RMID, NULL) < 0){                                  // remove the msg que, NULL means'no flags'
        report_and_exit("trouble removing queue...");
    }

    return 0;
}
```

compile & run msg que sender:
```text
$ gcc -o s sender.c && ./s
msg1 sent as type 1
msg2 sent as type 1
msg3 sent as type 2
msg4 sent as type 2
msg5 sent as type 3
msg6 sent as type 3
```

compile & run msg que receiver:
```text
$ gcc -o r receiver.c && ./c
msg5 received as type 3
msg1 received as type 1
msg3 received as type 2
msg2 received as type 1
msg6 received as type 3
msg4 received as type 2
```

the result above shows that, the message queue persists after the sender process exit.
the queue is destoryed only after receiver program explicitly remove it by msgctl syscall.

<hr>

### # appendix
message queues and pipes are typically unidirectional: one process writes and
another process reads. there existed bi-directional named pipes,
but ipc mechanism is best when its simplest.

<hr>

### # kafka message queue & tracing (todo)
