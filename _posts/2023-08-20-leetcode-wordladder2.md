---
layout: post
title: "word ladder II (leetcode 101 bfs build graph + backtracking)"
author: "melon"
date: 2023-08-20 22:40
categories: "2023"
tags:
  - leetcode
---

### # bfs + dfs algorithm

<hr>

### # theme description
leetcode 101 - 126 word ladder II 

<hr>

### # code
{% raw %}
```text
///usr/bin/env make ${0%%.cpp} CXXFLAGS="-g -Wall -Wextra -std=c++11 -O1" && exec ./${0%%.cpp}
// #include <bits/stdc++.h>
#include <vector>
#include <queue>
#include <string>
#include <unordered_set>
#include <unordered_map>
#include <iostream>

using namespace std;

template <typename T>
void print2dvec(const vector<vector<T>>& vec){
    for(auto subvec : vec){
        for_each(subvec.begin(), subvec.end(), [](T ele){ cout << ele << " "; });
        cout << endl;
    }
}

void backtracking(const string& src, const string& dst, unordered_map<string, vector<string>>& graph,
                  vector<string>& path, vector<vector<string>>& ans){
    if(src == dst){                                // end condition
        ans.push_back(path);
        return;
    }
    for(const string& str: graph[src]){
        path.push_back(str);                       // set
        backtracking(str, dst, graph, path, ans);  // start at cur str in graph
        path.pop_back();                           // unset
    }
}

vector<vector<string>> solve(string beginword, string endword, vector<string>& wordlist){
    vector<vector<string>> ans;
    unordered_set<string> wordset;                 // hold valid word path
    for(const string w : wordlist){                // build wordset (for easy search)
        wordset.insert(w);
    }
    if(!wordset.count(endword)){                   // if no endword, return
        return ans;
    }
    wordset.erase(beginword);                      // del start
    wordset.erase(endword);                        // del end
    unordered_set<string> head{beginword};
    unordered_set<string> tail{endword};
    unordered_map<string, vector<string>> graph;
    bool reversed = false;                         // control which side of graph(head/tail) to get updated
    bool found = false;
    while(!head.empty()){                          // if still has start point
        unordered_set<string> curstate;            // graph point of cur search level (for reverse check)
        for(const auto &w : head){                 // for each start str
            string s = w;                          // cache
            for(size_t i=0; i<s.size(); i++){      // for each char in str
                char ch = s[i];                    // set
                for(int j=0; j<26; j++){           // replace char with 26 alpha
                    s[i] = j + 'a';
                    if(tail.count(s)){             // if the generated str in tail
                        reversed? graph[s].push_back(w) : graph[w].push_back(s);
                        found = true;              // found a path to the end
                    }
                    if(wordset.count(s)){          // if generated str is a valid graph point 
                        reversed? graph[s].push_back(w) : graph[w].push_back(s);
                        curstate.insert(s);
                    }
                }
                s[i] = ch;                         // unset
            }
        }
        if(found){
            break;
        }
        for(const string w : curstate){
            wordset.erase(w);
        }
        if(curstate.size() <= tail.size()){        // if current building graph layer contains less ele than tail
            head = curstate;                       // continue search from q
        }
        else{                                      // else reverse search order
            reversed = !reversed;
            head = tail;                           // search from tail
            tail = curstate;
        }
    }
    if(found){                                     // backtracking to return the shortest path to destination
        vector<string> path = {beginword};
        backtracking(beginword, endword, graph, path, ans);
    }
    return ans;
}

int main(){
    // from begin -> end, each time 1 character change, and the middleware should be inside wordlist
    string beginword = "hit";
    string endword = "cog";
    vector<string> wordlist = {"hot", "dot", "dog", "lot", "log", "cog"};
    auto res = solve(beginword, endword,  wordlist);
    print2dvec(res);
    return 0;
}
```
output:
```txt
hit hot dot dog cog 
hit hot lot log cog 
```

{% endraw %}
