---
layout: post
title: "judge two directories are identical or not (python, helper script)"
author: "melon"
date: 2023-10-21 20:24
categories: "2023"
tags:
  - python
---

```text
import os
import hashlib

def get_file_hash(filepath):                             # compute & return file sha256 hash
    sha256 = hashlib.sha256()
    with open(filepath, 'rb') as f:
        while True:
            data = f.read(65536)
            if not data:
                break
            sha256.update(data)
    return sha256.hexdigest()

def compare_directories(dir1, dir2):
    dir1_hashes = set()                                  # set to store file hashes from dir1
    for root, dirs, files in os.walk(dir1):              # recursively walk through dir1 and calculate hashes
        for file in files:
            file_path = os.path.join(root, file)
            file_hash = get_file_hash(file_path)
            dir1_hashes.add(file_hash)
            # print(f"{file_path}: -----  {file_hash}")
    for root, dirs, files in os.walk(dir2):              # recursively walk through dir2 and compare hashes
        for file in files:
            file_path = os.path.join(root, file)
            file_hash = get_file_hash(file_path)
            # print(f"{file_path}: -----  {file_hash}")
            if file_hash not in dir1_hashes:
                return False
    return True

if __name__ == '__main__':
    dir1 = '/home/metung/nvim/config'
    dir2 = '/repo/metung/nvim/config'
    if compare_directories(dir1, dir2):
        print("all files are the same in the two directories.")
    else:
        print("not all files are the same in the two directories.")
```
