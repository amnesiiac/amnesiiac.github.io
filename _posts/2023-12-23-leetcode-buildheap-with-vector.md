---
layout: post
title: "build heap with vector (leetcode 101 heap)"
author: "melon"
date: 2023-12-23 11:47
categories: "2023"
tags:
  - leetcode
---

### # max/min heap implementation
heap is base data structure for priority queue
```text
///usr/bin/env make ${0%%.cpp} CXXFLAGS="-g -Wall -Wextra -std=c++11 -O1" && exec ./${0%%.cpp}
// #include <bits/stdc++.h>
#include <vector>
#include <iostream>
using namespace std;

template <typename T>
void print1dvec(const vector<T>& vec){
    for_each(vec.begin(), vec.end(), [](T ele){cout << ele << " "; });
    cout << endl;
}

template <typename T>
void print2dvec(const vector<T>& vec){
    for(auto subvec : vec){
        for_each(subvec.begin(), subvec.end(), [](T ele){cout << ele << " "; });
    }
    cout << endl;
}

template <typename T>
class heap{
public:
    vector<T> data;
    heap(){}
    heap(vector<T>& vec){                                  // init heap from a normal vector
        data = vec;
        buildheap(data);
    }
    void push(T k){                                        // push an ele into heap
        data.push_back(k);
        swim(data.size()-1);
    }
    void pop(){                                            // pop max value (first ele) from heap
        if(!data.empty()){
            data[0] = data.back();
            data.pop_back();
            sink(0);
        }
    }
    T top(){                                               // return max value of heap
        return data[0];
    }
    void heapsort(){                                       // sort data vec, min -> max
        for(int i=data.size()-1; i>=0; i--){               // for each possible heap len
            T tmp = data[0];                               // swap with first ele
            data[0] = data[i];
            data[i] = tmp;
            heapify(data, 0, i);                           // adjust subheap
        }
    }
private:
    void heapify(vector<T>& vec, int i, int n){            // vec[i] as root, vec[i]~vec[n-1] as a heap
        int l = 2*i + 1;
        int r = 2*i + 2;
        int larger = i;
        if(l<n && vec[l] > vec[larger]){                   // >: max top heap, <: min top heap
            larger = l;
        }
        if(r<n && vec[r] > vec[larger]){
            larger = r;
        }
        if(larger != i){                                   // swap vec[idx] with vec[larger]
            T tmp = vec[larger];
            vec[larger] = vec[i];
            vec[i] = tmp;
            heapify(vec, larger, n);
        }
    }
    void buildheap(vector<T>& vec){                        // build heap using aa vector
        int n = vec.size();
        int startidx = (n/2)-1;                            // index of last non-leaf node
        // for(int i=startidx; i>=0; i--){                 // reverse level order traversal start from last non-leaf node
        for(int i=n; i>=0; i--){                           // same as above, lack of efficiency
            heapify(vec, i, n);
        }
    }
    void swim(T pos){                                      // for push: new tailing node swim
        while((pos-1)/2>=0 && data[pos]>data[(pos-1)/2]){  // (pos-1)/2 = pos's parent
            T tmp = data[pos];
            data[pos] = data[(pos-1)/2];
            data[(pos-1)/2] = tmp;
            swim((pos-1)/2);
        }
    }
    void sink(T pos){                                      // for pop: new heading node sink
        T i = 2*pos+1;                                     // pos -> 2pos+1, 2pos+2 = child
        while(i < data.size()-1){
            if(data[i] < data[i+1]){
                ++i;
            }
            if(data[pos] >= data[i]){
                break;
            }
            T tmp = data[pos];
            data[pos] = data[i];
            data[i] = tmp;
            i = 2*i+1;
        }
    }
};

int main(){
    vector<int> vec = {19, 36, 17, 3, 25, 1, 2, 7, 100};
    heap<int> hh(vec);                                     // build heap using vec
    print1dvec(hh.data);                                   // min heap:   1 3 2 7 25 17 19 36 100
    hh.heapsort();
    print1dvec(hh.data);                                   // max to min: 100 36 25 19 17 7 3 2 1
    return 0;
}
```
