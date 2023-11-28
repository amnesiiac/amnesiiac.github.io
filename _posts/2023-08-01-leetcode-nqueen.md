---
layout: post
title: "n-queen (leetcode 101 backtracking)"
author: "melon"
date: 2023-08-01 15:13
categories: "2023"
tags:
  - leetcode
---

### # backtracking algorithm

<hr>

### # theme description

<hr>

### # code
{% raw %}
```text
///usr/bin/env make ${0%%.cpp} CXXFLAGS="-g -Wall -Wextra -std=c++11 -O1" && exec ./${0%%.cpp}
#include <bits/stdc++.h>
using namespace std;

// debug helper
void p1(vector<int>& vec){
    for_each(vec.begin(), vec.end(), [](int i){std::cout << i << " ";});
}

void p2(vector<vector<string>>& vecc){
    for(auto vec: vecc){
        for_each(vec.begin(), vec.end(), [](string i){cout << i << endl;});
        cout << endl;
    }
}

// recursive backtracking
void solve(vector<vector<string>>& ans, vector<string>& board, vector<bool>& col,
           vector<bool>& ldiag, vector<bool>& rdiag, int row, int n){
    if(row == n){                                       // meet the need
        ans.push_back(board);
        return;
    }
    for(int i=0; i<n; i++){                             // for each col
        if(col[i] || ldiag[i-row+n-1] || rdiag[i+row]){ // skip condition
            continue;
        }
        board[row][i] = 'Q';                            // set
        col[i] = true;    

        ldiag[i-row+n-1] = true;                        // y-x+n-1 = ldiag line number 
        rdiag[i+row] = true;                            // y+x = rdiag line number
        solve(ans, board, col, ldiag, rdiag, row+1, n); // recursive

        board[row][i] = '.';                            // reset
        col[i] = false;
        ldiag[i-row+n-1] = false;
        rdiag[i+row] = false;
    }
}
int main(){
    int n = 4;
    vector<vector<string>> ans;
    vector<string> board(n, string(n, '.'));
    vector<bool> col(n, false);
    vector<bool> ldiag(2*n-1, false);
    vector<bool> rdiag(2*n-1, false);
    solve(ans, board, col, ldiag, rdiag, 0, n);
    p2(ans);
    return 0;
}
```
output:
```txt
.Q..
...Q
Q...
..Q.

..Q.
Q...
...Q
.Q..
```

{% endraw %}
