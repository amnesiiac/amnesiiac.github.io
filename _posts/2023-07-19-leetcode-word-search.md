---
layout: post
title: "word search (leetcode 101 backtracking)"
author: "melon"
date: 2023-07-19 20:58
categories: "2023"
tags:
  - leetcode
---

### # backtracking algorithm

<hr>

### # theme description
79

<hr>

### # code
{% raw %}
```text
///usr/bin/env make ${0%%.cpp} CXXFLAGS="-g -Wall -Wextra -std=c++11 -O1" && exec ./${0%%.cpp}
#include <bits/stdc++.h>
// #include <vector>
// #include <string>
// #include <iostream>
using namespace std;


void wordsearch(vector<vector<char>>& board, const string& word, vector<vector<bool>>& searched,
                int i, int j, int pos, bool& ret){
    if(i<0 || j<0 || i>=board.size() || j>=board[0].size()){ // if out range
        return;
    }
    if(searched[i][j] == 1){ // if i,j already seached
        return;
    }
    if(ret){ // if already meet the need, then just abort continue searches
        return;
    }
    if(board[i][j] != word[pos]){ // if cur i,j is not meet the need
        return;
    }
    if(pos == word.size()-1){ // if num of char meet the need is same as word, and ret is still false
        ret = true;
        return;
    }
    // mark as searched
    searched[i][j] = 1;
    // search 4 directions
    wordsearch(board, word, searched, i+1, j, pos+1, ret);
    wordsearch(board, word, searched, i-1, j, pos+1, ret);
    wordsearch(board, word, searched, i, j+1, pos+1, ret);
    wordsearch(board, word, searched, i, j-1, pos+1, ret);
    // restore i,j as not searched -> for new root search to begin
    searched[i][j] = 0;
}

int main(){
    // string word = "ABFCED";  // existed
    string word = "ABFCEF";  // non-existed
    bool ret = false;
    vector<vector<char>> board = {{'A', 'B', 'C', 'E'}, {'S', 'F', 'C', 'S'}, {'A', 'D', 'E', 'E'}};
    vector<vector<bool>> searched(board.size(), vector<bool>(board[0].size()));
    if(board.empty()){ // no word is in!
        return 0;
    }
    // for each possible starting point -> start backtracking search
    for(int i=0; i<board.size(); i++){
        for(int j=0; j<board[0].size(); j++){
            wordsearch(board, word, searched, i, j, 0, ret);
        }
    }
    if(ret){
        cout << "the word " << word << " is in board!" << endl;
    }
    else{
        cout << "the word is not in board" << endl;
    }
    return 0;
}
```
output:
```txt
the word ABFCED is in board!
```

{% endraw %}
