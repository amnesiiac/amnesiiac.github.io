---
layout: post
title: "set -u with unset variables (shell)"
author: "melon"
date: 2023-12-26 20:29
categories: "2023"
tags:
  - shell
---

### # using unset variables with set -u enable
test code sample 1:
```text
#!/bin/sh
set -eu -o pipefail                        # set -u enabled

set_basebranch() {
    if [ -n "${CI_COMMIT_BRANCH}" ]; then  # unset shell variable
        echo "test"
    fi
}

readonly BASEBRANCH=$(set_basebranch)
echo "$BASEBRANCH"

# output:
# ./h.sh: line x: CI_COMMIT_BRANCH: unbound variable
```
reason of failure: set -u will abort exactly when reference a variable not set yet.

fix code for code sample 1:
```text
#!/bin/sh
set -eu -o pipefail                                # set -u enabled

set_basebranch() {
    if [ -n "${CI_COMMIT_BRANCH:-reborn}" ]; then  # correct
        :
    fi
}

readonly BASEBRANCH=$(set_basebranch)
echo "$BASEBRANCH"
```

test code sample 2:
```text
#!/bin/sh
set -eu -o pipefail                                # set -u enable

set_basebranch() {
    if [ -n "${CI_COMMIT_BRANCH:-reborn}" ]; then
        echo "${CI_COMMIT_BRANCH}"                 # unbound variable?!
    fi
}

readonly BASEBRANCH=$(set_basebranch)
echo "$BASEBRANCH"

# ./h.sh: line x: CI_COMMIT_BRANCH: unbound variable
```
reason of failure: the default value only work for once and the next same unset variable should also use default value.

fix code for code sample 2:
```text
#!/bin/sh
set -eu -o pipefail                                # set -u enable

set_basebranch() {
    if [ -n "${CI_COMMIT_BRANCH:-reborn}" ]; then
        echo "${CI_COMMIT_BRANCH:-reborn}"         # correct
    fi
}

readonly BASEBRANCH=$(set_basebranch)
echo "$BASEBRANCH"
```

but normally we dont use like the above, a more concise way is:
```text
#!/bin/sh
set -eu -o pipefail                                # set -u enable

ci_commit_branch="${CI_COMMIT_BRANCH:-reborn}"     # use replica variable

set_basebranch() {
    if [ -n "${ci_commit_branch}" ]; then
        echo "${ci_commit_branch}"
    fi
}

readonly BASEBRANCH=$(set_basebranch)
echo "$BASEBRANCH"
```

<hr>

### # todo
ref: https://www.baeldung.com/linux/bash-variable-assign-default
