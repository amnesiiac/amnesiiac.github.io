---
layout: post
title: "max area of islands (leetcode 101 dfs stack)"
author: "melon"
date: 2023-05-15 21:26
categories: "2023"
tags:
  - leetcode
---

### # max area of islands (dfs) - stack version
using dfs implemented by stack to compute the "local area" begining with each start point.
{% raw %}
```cpp
///usr/bin/env make ${0%%.cpp} CXXFLAGS="-g -Wall -Wextra -std=c++11 -O1" && exec ./${0%%.cpp}
#include <bits/stdc++.h>
using namespace std;

vector<int> directions = {-1, 0, 1, 0, -1};

int max_area_of_islands(vector<vector<int>> &vec){
    int m=vec.size(); int n=m? vec[0].size():0;
    int area=0; int local_area=0; int x, y;
    for(int i=0; i<m; i++){
        for(int j=0; j<n; j++){
            if(vec[i][j]){// if meet a startable point
                local_area = 1; // init local_area
                vec[i][j] = 0; // set it not startable any more
                stack<pair<int, int>> islands;
                islands.push(make_pair(i, j));
                while(!islands.empty()){// dfs to compute local area started from (i,j)
                    // auto [r, c] = islands.top();// structured binding -> introduced in c++17
                    int r = islands.top().first; int c = islands.top().second;
                    islands.pop(); // pop dealt element out
                    for(int k=0; k<4; k++){// navigate 4 direction
                        x = r+directions[k]; y = c+directions[k+1]; // move 1 step
                        // note: the vec[x][y] should be placed the last or will lead to seg fault
                        if(x>=0 && x<m && y>=0 && y<n && vec[x][y]){  // check boundary
                            local_area+=1;
                            islands.push(make_pair(x, y));
                            vec[x][y]=0; // set it not startable any more
                        }
                    }
                }
                area = max(area, local_area); // refresh max area
            }
        }
    }
    return area;
}

int main(){
    vector<vector<int>> vec = {{1,0,1,1,0,1,0,1}, {1,0,1,1,0,1,1,1}, {0,0,0,0,0,0,0,1}};
    std::cout << "the max area of islands in the map is " << max_area_of_islands(vec) << std::endl;
    return 0;
}
```
```txt
the max area of islands in the map is 6
```

<hr>

### # max area of islands (dfs) - recursive version
using recursive dfs to compute the "local area" begining with each start point.
```cpp
///usr/bin/env make ${0%%.cpp} CXXFLAGS="-g -Wall -Wextra -std=c++11 -O1" && exec ./${0%%.cpp}
// max area of islands (dfs) - stack version
#include <bits/stdc++.h>
using namespace std;

vector<int> directions = {-1, 0, 1, 0, -1};

// in: start point, out: area when begin with this start point
int dfs(vector<vector<int>> &vec, int r, int l){
    if(vec[r][l]==0){return 0;}// if already searched or just water
    vec[r][l] = 0; // marked as searched
    int area = 1;// startup
    int x, y;
    for(int k=0; k<4; k++){
        x = r+directions[k]; y=l+directions[k+1];
        if(x>=0 && x<vec.size() && y>=0 && y<vec[0].size()){ // if inside scope
            area+=dfs(vec, x, y); // recursively compute
        }
    }
    return area;
}

int max_area_of_islands(vector<vector<int>> &vec){
    int m=vec.size(); int n=m? vec[0].size():0; int area=0;
    for(int i=0; i<m; i++){
        for(int j=0; j<n; j++){
            if(vec[i][j]){// if can started here
                area = max(area, dfs(vec, i, j));
            }
        }
    }
    return area;
}

// the max area of islands in the map is 6
int main(){
    vector<vector<int>> vec = {{1,0,1,1,0,1,0,1}, {1,0,1,1,0,1,1,1}, {0,0,0,0,0,0,0,1}};
    std::cout << "the max area of islands in the map is " << max_area_of_islands(vec) << std::endl;
    return 0;
}
```
```txt
g++ -g -Wall -Wextra -std=c++11 -O1    695-1.cpp   -o 695-1
695-1.cpp: In function ‘int dfs(std::vector<std::vector<int> >&, int, int)’:
695-1.cpp:15:31: warning: comparison between signed and unsigned integer expressions [-Wsign-compare]
if(x>=0 && x<vec.size() && y>=0 && y<vec[0].size()){ // if inside scope
                      ^
695-1.cpp:15:58: warning: comparison between signed and unsigned integer expressions [-Wsign-compare]
if(x>=0 && x<vec.size() && y>=0 && y<vec[0].size()){ // if inside scope
                                                 ^
the max area of islands in the map is 6
```
{% endraw %}
