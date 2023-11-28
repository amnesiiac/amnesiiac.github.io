---
layout: post
title: "num of friend circles (leetcode 101 dfs stack)"
author: "twistfatezz"
date: 2023-05-20 09:34
categories: "2023"
tags:
  - leetcode
---

### # theme description
Given a two-dimensional 0-1 matrix, if the (i, j)th position is 1, it means that the i-th person and the j-th person are friends.  
The friendship can be transmitted, that is, if a is b's Friends, b is a friend of c, then a and c are also friends, in other words, these three people are in the same circle of friends.  
Find out how many circles of friends there are in total.

<hr>

### # num of friend circles (dfs) - stack version
{% raw %}
```cpp
///usr/bin/env make ${0%%.cpp} CXXFLAGS="-g -Wall -Wextra -std=c++11 -O1" && exec ./${0%%.cpp}
#include <vector>
#include <iostream>
#include <stack>
using namespace std;

int friend_circle(vector<vector<int>> &vec){
    int n = vec.size();
    int num_friend_circle=0;
    for(int i=0; i<n; i++){ // for each person
        if(vec[i][i]==1){ // if the person not checked 
            num_friend_circle++;
            stack<int> member;
            member.push(i); // init
            vec[i][i]=0; // set current person checked
            while(!member.empty()){ // start dfs search
                int j=0; i=member.top();
                for(int j=0; j<n; j++){ // iterate all possible friend
                    if(j==i){
                        continue;
                    }
                    if(vec[i][j]==1){ // (i,j) are friend
                        vec[i][j]=0; vec[j][i]=0; // set them checked
                        vec[j][j]=0; // set self checked
                        member.push(j); // push to enable dfs further
                    }
                }
                member.pop(); // pop the person out if already check all the relationship
            }
        }
    }
    return num_friend_circle;
} 

int main(){
    vector<vector<int>> vec1 = {{1,1,0},{1,1,0},{0,0,1}};
    vector<vector<int>> vec2 = {{1,1,1,1},{1,1,0,1},{1,0,1,0},{1,1,0,1}};
    vector<vector<int>> vec3 = {{1,1,0,1},{1,1,0,1},{0,0,1,0},{1,1,0,1}};
    vector<vector<int>> vec4 = {{1,0,0,0},{0,1,0,0},{0,0,1,0},{0,0,0,1}};
    std::cout << "num of friend circle is: " << friend_circle(vec1) << std::endl;
    std::cout << "num of friend circle is: " << friend_circle(vec2) << std::endl;
    std::cout << "num of friend circle is: " << friend_circle(vec3) << std::endl;
    std::cout << "num of friend circle is: " << friend_circle(vec4) << std::endl;
    return 0;
}
```
```txt
test samples:
1 1 0     1 1 1 1     1 1 0 1     1 0 0 0
1 1 0     1 1 0 1     1 1 0 1     0 1 0 0
0 0 1     1 0 1 0     0 0 1 0     0 0 1 0
          1 1 0 1     1 1 0 1     0 0 0 1
```
```txt
test result:
num of friend circle is: 2
num of friend circle is: 1
num of friend circle is: 2
num of friend circle is: 4
```

<hr>

### # max area of islands (dfs) - recursive version
```cpp
///usr/bin/env make ${0%%.cpp} CXXFLAGS="-g -Wall -Wextra -std=c++11 -O1" && exec ./${0%%.cpp}
#include <vector>
#include <iostream>
#include <stack>
using namespace std;

void dfs(vector<vector<int>> &vec, int i){ // check all friend circle of person i
    // no recursive end condition needed
    for(int j=0; j<vec.size(); j++){
        if(j==i){
            continue;
        }
        if(vec[i][j]==1){
            vec[i][j]=0; vec[j][i]=0;
            vec[j][j]=0;
            dfs(vec, j); // recursively check all friend circle of j
        }
    }
}

int friend_circle(vector<vector<int>> &vec){
    int n = vec.size();
    int num_friend_circle=0;
    for(int i=0; i<n; i++){
        if(vec[i][i]==1){ // if not checked
            vec[i][i]=0; // set as checked
            dfs(vec, i); // check friend circle of i, mask all friend circle as 0 (checked)
            num_friend_circle++;
        }
    }
    return num_friend_circle;
} 

int main(){
    vector<vector<int>> vec1 = {{1,1,0},{1,1,0},{0,0,1}};
    vector<vector<int>> vec2 = {{1,1,1,1},{1,1,0,1},{1,0,1,0},{1,1,0,1}};
    vector<vector<int>> vec3 = {{1,1,0,1},{1,1,0,1},{0,0,1,0},{1,1,0,1}};
    vector<vector<int>> vec4 = {{1,0,0,0},{0,1,0,0},{0,0,1,0},{0,0,0,1}};
    std::cout << "num of friend circle is: " << friend_circle(vec1) << std::endl;
    std::cout << "num of friend circle is: " << friend_circle(vec2) << std::endl;
    std::cout << "num of friend circle is: " << friend_circle(vec3) << std::endl;
    std::cout << "num of friend circle is: " << friend_circle(vec4) << std::endl;
    return 0;
}
```
```txt
num of friend circle is: 2
num of friend circle is: 1
num of friend circle is: 2
num of friend circle is: 4
```
{% endraw %}
