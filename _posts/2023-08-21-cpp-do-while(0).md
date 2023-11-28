---
layout: post
title: "do while(0) (c++)"
author: "melon"
date: 2023-08-21 21:21
categories: "2023"
tags:
  - cpp
---

### # why do we need do while(0) when defining macros
given a normal macro definition as follows:
```text
#define foo(x) bar(x); baz(x)
```

1) expected macro expansion:
```
foo(wolf);                 // given src:
bar(wolf); baz(wolf);      // expanded result (expected)
```

2) unexpected macro expansion:
```
if(!feral)                 // given src:
    foo(wolf);

if(!feral)                 // macro expanded result (unexpected)
    bar(wolf);
baz(wolf);
```

3) unexpected macro expansion:
```
if(!feral)                 // given src:
    foo(wolf);
else
    bin(wolf);

if(!feral)                 // macro expanded result (unexpected: syntax error)
    bar(wolf); baz(wolf);
else
    bin(wolf);
```

it's difficult to write multistatement macros the work as expected in all situations,
but we could use trick like the following:
```text
#define foo(x) do{bar(x); baz(x);} while(0)
```

4) robust expansion case:
```
if(!feral)                         // given src:
    foo(wolf);
if(!feral){                        // macro expanded result (expected)
    do{bar(x); baz(x);} while(0)
}
if(!feral){                        // equivalent format: expanded result (expected)
    bar(x); baz(x);
}
```

<hr>

### # do while(0) to enclose multiline statement
```text
#define foo \
	do{ \
		statement_1; \
		statement_2; \
	}while(0) \
```

<hr>

### # code in action
SAFE_CLOSESOCKET(fd) definition
```text
// libhv/base/hsocket.h
#ifndef SAFE_CLOSESOCKET
#define SAFE_CLOSESOCKET(fd) \ 
    do{ \
        if((fd) >= 0){ \
        closesocket(fd); (fd) = -1; \
        } \
    }while(0) \
#endif
```

<hr>

### # ref
ref: https://www.pixelstech.net/article/1390482950-do-%7b-%7d-while-(0)-in-macros  
ref: https://www.bruceblinn.com/linuxinfo/DoWhile.html
