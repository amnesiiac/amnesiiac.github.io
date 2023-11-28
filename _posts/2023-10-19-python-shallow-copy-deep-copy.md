---
layout: post
title: "deep copy & shallow copy (python)"
author: "melon"
date: 2023-10-19 22:24
categories: "2023"
tags:
  - python
---

### # memory model of refrerence & deep copy & shallow copy
```txt
                                                ┌───┬───┐
                                       ┌--------+ 3 │ 5 │
     ┌───────────────┐                 |        └─+─┴───┘
     │┌─────────────┐│       ┌───┬───┬─┼─┬───┐    |
     ││lst: original├┼-------+ 1 │ 2 │ + │ 4 │    |
     │└─────────────┘│       └─+─┴───┴───┴───┘    |
     │┌─────────────┐│         |                  |
     ││reflst       ├┼---------┘       ┌----------┘
     │└─────────────┘│                 | 
     │┌─────────────┐│       ┌───┬───┬─┼─┬───┐                   ┌───┬───┐
     ││copylst      ├┼-------+ 1 │ 2 │ + │ 4 │                   │ 3 │ 5 │
     │└─────────────┘│       └───┴───┴───┴───┘                   └─+─┴───┘
     │┌─────────────┐│                                   ┌───┬───┬─┼─┬───┐
     ││deepcopylst  ├┼-----------------------------------+ 1 │ 2 │ + │ 4 │
     │└─────────────┘│                                   └───┴───┴───┴───┘
     └───────────────┘

```
1) the original list and reflst share completely the same mem obj.  
2) the copy.copy list and original list share a different base mem obj, but the same sub mem obj.  
3) the copy.deepcopy list and original list has nothing overlapped.

<hr>

### # sample code for illustration
1) reference: all the changes are synchronized.
```text
#!/usr/bin/env python3
import copy

lst = [1, 2, [3, 5], 4]            # original  id: 4515770304  value: [1, 2, [3, 5], 4]
reflst = lst                       # reference id: 4515770304  value: [1, 2, [3, 5], 4]

                                                                          *
reflst[1] = 7                      # original  id: 4515770304  value: [1, 7, [3, 5], 4] -> changed
                                   # reference id: 4515770304  value: [1, 7, [3, 5], 4]

                                                                              *
reflst[2][0] = 7                   # original  id: 4515770304  value: [1, 7, [7, 5], 4] -> changed
                                   # reference id: 4515770304  value: [1, 7, [7, 5], 4]

                                                                              *  *
reflst[2] = [0, 0]                 # original  id: 4515770304  value: [1, 7, [0, 0], 4] -> changed
                                   # reference id: 4515770304  value: [1, 7, [0, 0], 4]
```

2) shallow copy (copy.copy): not all changes are synchronized.
```text
#!/usr/bin/env python3
import copy

lst = [1, 2, [3, 5], 4]            # original     id: 4452887616  value: [1, 2, [3, 5], 4]
copylst = copy.copy(lst)           # shallowcopy  id: 4452887744  value: [1, 2, [3, 5], 4]

                                                                              *
copylst[1] = 7                     # original     id: 4452887616  value: [1, 2, [3, 5], 4] -> not changed
                                   # shallowcopy  id: 4452887744  value: [1, 7, [3, 5], 4]

                                                                                  *
copylst[2][0] = 7                  # original     id: 4452887616  value: [1, 2, [7, 5], 4] -> changed
                                   # shallowcopy  id: 4452887744  value: [1, 7, [7, 5], 4]

                                                                                  *  *
copylst[2] = [0, 0]                # original     id: 4452887616  value: [1, 2, [7, 5], 4] -> not changed
                                   # shallowcopy  id: 4452887744  value: [1, 7, [0, 0], 4]
```

3) deep copy (copy.deepcopy): all changes are not synchronized.
```text
#!/usr/bin/env python3
import copy

lst = [1, 2, [3, 5], 4]            # original  id: 4374326976  value: [1, 2, [3, 5], 4]
deepcopylst = copy.deepcopy(lst)   # deepcopy  id: 4374327104  value: [1, 2, [3, 5], 4]

                                                                          *
deepcopylst[1] = 7                 # original  id: 4374326976  value: [1, 2, [3, 5], 4] -> not changed
                                   # deepcopy  id: 4374327040  value: [1, 7, [3, 5], 4]

                                                                              *
deepcopylst[2][0] = 7              # original  id: 4374326976  value: [1, 2, [3, 5], 4] -> not changed
                                   # deepcopy  id: 4374327040  value: [1, 7, [7, 5], 4]

                                                                              *  *
deepcopylst[2] = [0, 0]            # original  id: 4374326976  value: [1, 2, [3, 5], 4] -> not changed
                                   # deepcopy  id: 4374327040  value: [1, 7, [0, 0], 4]
```
