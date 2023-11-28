---
layout: post
title: "switch structure (shell)"
author: "twistfatezz"
date: 2023-04-02 17:48
categories: "2023"
tags:
  - shell
---

switch structure in shell is like:
```shell
# case structure
case_no=2
case case_no in
    1 )
        echo "case_structure: 1st condition"  # command
    ;;
    2 )  # condition
        echo "case_structure：2nd condition"
    ;;
    case_no )
        echo "case_structure: 3nd condition"
    ;;
esac
```

