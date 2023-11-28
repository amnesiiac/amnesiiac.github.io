---
layout: post
title: "minimum window substring (leetcode 101 double ptr)"
author: "twistfatezz"
date: 2023-04-12 18:50
categories: "2023"
tags:
  - leetcode 
---
### # double pointer algorithm
```cpp
#include <bits/stdc++.h>
using namespace std;

string minwin(string &s, string &t){
    vector<int> chars(128, 0);      // number of char needed
    vector<bool> flag(128, false);  // if a char inside t
    for(int x=0; x<t.size(); x++){  // init
        flag[t[x]] = true;
        ++chars[t[x]];
    }
    int r=0; int l=0;           // right/left pointer
    int accepted=0;             // number of chars meet the need (currently)
    int minl = 0;               // left pointer of minmum window
    int minsize = s.size()+1;   // minmum window size
    for(r=0; r<s.size(); r++){
        if(flag[s[r]]){         // if current char in t
            chars[s[r]]--;      // update char needed
            if(chars[s[r]]>=0){ // if the char is accepted as substring
                ++accepted;
            }
            while(accepted== t.size()){  // if all char(t) is collected in s
                if(r-l+1 < minsize){     // update minl/minsize 
                    minl = l;
                    minsize = r-l+1;
                }
                ++chars[s[l]];                     // update chars needed
                if(flag[s[l]] && chars[s[l]]>0){  // if the last char(=0) accepted w.r.t each r
                    --accepted;                    // break loop for l
                }
                ++l;
            }
        }
    }
    return minsize<s.size()? s.substr(minl, minsize):"none"; // if no substr comform
}

int main(){
    // 1
    string s1 = "EBACDEFBEAC";
    string t1 = "ABC";
    // 2
    string s2 = "ADOBECODEBANC";
    string t2 = "ABC";
    // 3
    string s3 = "ADOBEDODEBANFDEFBAG";
    string t3 = "ABC";
    std::cout << "minmum substr of (s1,t1): " << minwin(s1, t1) << std::endl;
    std::cout << "minmum substr of (s2,t2): " << minwin(s2, t2) << std::endl;
    std::cout << "minmum substr of (s3,t3): " << minwin(s3, t3) << std::endl;
    return 0;
}
```
```txt
# output:
minmum substr of (s1,t1): BAC
minmum substr of (s2,t2): BANC
minmum substr of (s3,t3): none
```
