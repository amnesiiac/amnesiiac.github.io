---
layout: post
title: "quick sort algorithm (leetcode 101)"
author: "twistfatezz"
date: 2023-04-20 23:03
categories: "2023"
tags:
  - leetcode
---

### # quicksort algorithm 
1: return the arr if has less than 1 element. <br>
2: select a pivot (usually the first), the pivot selection can impact the algorithm performance. <br>
3: compare && swap: partition the array into 2 subarr: one with elements <= pivot, and another with elements > pivot. <br>
4: recursively quicksort the 2 subarr. <br>
5: concatenate the 2 subarr with the pivot to produce the sorted array.

<hr>

### # performance analysis 
• Unstable: does not preserve the relative order of equal elements compare to the init array. <br>
• The worst time complexity: O(n^2), for each compare in while loop, a swap is needed. <br>
• The average time complexity: O(nlog2n), there are n elements in the array for compare & swap, and binary division algorithm has log2n to divide all elements into pair. <br>
• The average space complexity: O(log2n), the algorithm sort array in place (constant space), and there are totally [log2n, n] recursion tree layer.

<hr>

### # quicksort cpp implementation
{% raw %}
method-1: select && swap.
```cpp
#include <vector>
#include <iostream>
using namespace std;

void quicksort(int arr[], int start, int end){
    if(arr==nullptr || start<0 || start>=end){
        return;
    }
    int i=start; int j=end; int base=arr[start];// selection of base will impact the performance 
    while(i<j){
        while(arr[j]>=base && i<j){
            j--;
        }
        while(arr[i]<=base && i<j){
            i++;
        } 
        int tmp0=arr[i]; arr[i]=arr[j]; arr[j]=tmp0; // swap -> unstable
    }
    int tmp1=arr[i]; arr[i]=base; arr[start]=tmp1; // swap -> set base in the 'middle'
    quicksort(arr, start, i-1);
    quicksort(arr, i+1, end);
}
```
method-2: filling the holes.
```cpp
#include <vector>
#include <iostream>
using namespace std;

void quicksort(vector<int> &nums, int l, int r){
    if(l+1 >= r){return;}
    int first = l, last = r - 1, base = nums[first]; // select && save the init hole
    while (first < last){
        while(first < last && nums[last] >= base){
            --last;
        }
        nums[first] = nums[last]; // dig out the last && insert it in hole first
        while(first < last && nums[first] <= base){
            ++first;
        }
        nums[last] = nums[first]; // dig out the first && insert in hole last
    }
    nums[first] = base; // recover the init hole
    quicksort(nums, l, first);
    quicksort(nums, first + 1, r);
}
```
```cpp
int main(){
    vector<int> vec = {6, 3, 5, 1, 2, 10, 5, 7};
    quicksort(vec, 0, vec.size());
    for_each(vec.begin(), vec.end(), [](int i){std::cout << i << " ";});
    return 0;
}
```
```txt
# output:
1 2 3 5 5 6 7 10
```

{% endraw %}
