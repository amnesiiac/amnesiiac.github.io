---
layout: post
title: "Region proposal methods"
subtitle: '用于目标检测的区域建议算法简介|代码解析'
author: "twistfatezz"
header-style: text
# header-img: "img/post-bg-alitrip.jpg"
# header-mask: 0.1
mathjax: true
date: 2019-10-01 21:23 
lang: ch 
catalog: true 
categories: documentation
tags:
  - nn toolkit
  - Time 2019
---

### Introduction

### Selective search
对于selective search算法原理想要深入了解的移步：[selective search official blog](https://www.koen.me/research/selectivesearch/)。官方博客中两篇论文：[Selective Search for Object Recognition](https://ivi.fnwi.uva.nl/isis/publications/2013/UijlingsIJCV2013/UijlingsIJCV2013.pdf)和[Segmentation as Selective Search for Object Recognition](https://ivi.fnwi.uva.nl/isis/publications/2011/vandeSandeICCV2011/vandeSandeICCV2011.pdf)。

另外如果想要简单的了解selective search的基本算法，请看[这篇总结](https://www.learnopencv.com/selective-search-for-object-detection-cpp-python/)，以及[这篇总结](https://www.cnblogs.com/zyly/p/9259392.html)。

### Code
python可以通过如下的方式安装selective search包
```shell
pip install selectivesearch
```
以下为上述包中的源码，长度不长，贴出来方便分析。
```python
# -*- coding: utf-8 -*-
import skimage.io
import skimage.feature
import skimage.color
import skimage.transform
import skimage.util
import skimage.segmentation
import numpy

# "Selective Search for Object Recognition" by J.R.R. Uijlings et al.
#  - Modified version with LBP extractor for texture vectorization

def _generate_segments(im_orig, scale, sigma, min_size):
    """
        segment smallest regions by the algorithm of Felzenswalb and Huttenlocher
    """
    # open the Image
    im_mask = skimage.segmentation.felzenszwalb(skimage.util.img_as_float(im_orig), 
                                      scale=scale, sigma=sigma, min_size=min_size)
    # merge mask channel to the image as a 4th channel
    im_orig = numpy.append(im_orig, numpy.zeros(im_orig.shape[:2])[:, :, numpy.newaxis], axis=2)
    im_orig[:, :, 3] = im_mask
    return im_orig


def _sim_colour(r1, r2):
    """
        calculate the sum of histogram intersection of colour
    """
    return sum([min(a, b) for a, b in zip(r1["hist_c"], r2["hist_c"])])


def _sim_texture(r1, r2):
    """
        calculate the sum of histogram intersection of texture
    """
    return sum([min(a, b) for a, b in zip(r1["hist_t"], r2["hist_t"])])


def _sim_size(r1, r2, imsize):
    """
        calculate the size similarity over the image
    """
    return 1.0 - (r1["size"] + r2["size"]) / imsize


def _sim_fill(r1, r2, imsize):
    """
        calculate the fill similarity over the image
    """
    bbsize = ((max(r1["max_x"], r2["max_x"]) - min(r1["min_x"], r2["min_x"]))
        * (max(r1["max_y"], r2["max_y"]) - min(r1["min_y"], r2["min_y"])))
    return 1.0 - (bbsize - r1["size"] - r2["size"]) / imsize


def _calc_sim(r1, r2, imsize):
    return (_sim_colour(r1, r2) + _sim_texture(r1, r2)
            + _sim_size(r1, r2, imsize) + _sim_fill(r1, r2, imsize))


def _calc_colour_hist(img):
    """
        Calculate colour histogram for each region.
        The size of output histogram will be BINS * COLOUR_CHANNELS(3).
        Number of bins is 25 as same as [uijlings_ijcv2013_draft.pdf].
        extract HSV.
    """
    BINS = 25
    hist = numpy.array([])
    for colour_channel in (0, 1, 2):
        # extracting one colour channel
        c = img[:, colour_channel]
        # calculate histogram for each colour and join to the result
        hist = numpy.concatenate([hist] + [numpy.histogram(c, BINS, (0.0, 255.0))[0]])
    # L1 normalize
    hist = hist / len(img)
    return hist


def _calc_texture_gradient(img):
    """
        Calculate texture gradient for entire image.
        The original SelectiveSearch algorithm proposed Gaussian derivative
        for 8 orientations, but we use LBP instead.
        Output will be [height(*)][width(*)]
    """
    ret = numpy.zeros((img.shape[0], img.shape[1], img.shape[2]))
    for colour_channel in (0, 1, 2):
        ret[:, :, colour_channel] = skimage.feature.local_binary_pattern(
            img[:, :, colour_channel], 8, 1.0)
    return ret


def _calc_texture_hist(img):
    """
        Calculate texture histogram for each region
        Calculate the histogram of gradient for each colours
        The size of output histogram will be BINS * ORIENTATIONS * COLOUR_CHANNELS(3)
    """
    BINS = 10
    hist = numpy.array([])
    for colour_channel in (0, 1, 2):
        # mask by the colour channel
        fd = img[:, colour_channel]
        # calculate histogram for each orientation and concatenate them all
        # and join to the result
        hist = numpy.concatenate([hist] + [numpy.histogram(fd, BINS, (0.0, 1.0))[0]])
    # L1 Normalize
    hist = hist / len(img)
    return hist


def _extract_regions(img):
    R = {}
    # get hsv image
    hsv = skimage.color.rgb2hsv(img[:, :, :3])
    # pass 1: count pixel positions
    for y, i in enumerate(img):
        for x, (r, g, b, l) in enumerate(i):
            # initialize a new region
            if l not in R:
                R[l] = {"min_x": 0xffff, "min_y": 0xffff, 
                        "max_x": 0, "max_y": 0, "labels": [l]}
            # bounding box
            if R[l]["min_x"] > x:
                R[l]["min_x"] = x
            if R[l]["min_y"] > y:
                R[l]["min_y"] = y
            if R[l]["max_x"] < x:
                R[l]["max_x"] = x
            if R[l]["max_y"] < y:
                R[l]["max_y"] = y
    # pass 2: calculate texture gradient
    tex_grad = _calc_texture_gradient(img)
    # pass 3: calculate colour histogram of each region
    for k, v in R.items():
        # colour histogram
        masked_pixels = hsv[:, :, :][img[:, :, 3] == k]
        R[k]["size"] = len(masked_pixels / 4)
        R[k]["hist_c"] = _calc_colour_hist(masked_pixels)
        # texture histogram
        R[k]["hist_t"] = _calc_texture_hist(tex_grad[:, :][img[:, :, 3] == k])
    return R


def _extract_neighbours(regions):
    def intersect(a, b):
        if (a["min_x"] < b["min_x"] < a["max_x"]
                and a["min_y"] < b["min_y"] < a["max_y"]) or (
            a["min_x"] < b["max_x"] < a["max_x"]
                and a["min_y"] < b["max_y"] < a["max_y"]) or (
            a["min_x"] < b["min_x"] < a["max_x"]
                and a["min_y"] < b["max_y"] < a["max_y"]) or (
            a["min_x"] < b["max_x"] < a["max_x"]
                and a["min_y"] < b["min_y"] < a["max_y"]):
            return True
        return False

    R = regions.items()
    r = [elm for elm in R]
    R = r
    neighbours = []
    for cur, a in enumerate(R[:-1]):
        for b in R[cur + 1:]:
            if intersect(a[1], b[1]):
                neighbours.append((a, b))
    return neighbours


def _merge_regions(r1, r2):
    new_size = r1["size"] + r2["size"]
    rt = {
        "min_x": min(r1["min_x"], r2["min_x"]),
        "min_y": min(r1["min_y"], r2["min_y"]),
        "max_x": max(r1["max_x"], r2["max_x"]),
        "max_y": max(r1["max_y"], r2["max_y"]),
        "size": new_size,
        "hist_c": (r1["hist_c"] * r1["size"] + r2["hist_c"] * r2["size"]) / new_size,
        "hist_t": (r1["hist_t"] * r1["size"] + r2["hist_t"] * r2["size"]) / new_size,
        "labels": r1["labels"] + r2["labels"]
    }
    return rt


def selective_search(
        im_orig, scale=1.0, sigma=0.8, min_size=50):
    '''
    Parameters
    ----------
        im_orig: ndarray - Input image
        scale: int - Free parameter. Higher means larger clusters in felzenszwalb segmentation.
        sigma: float - Width of Gaussian kernel for felzenszwalb segmentation.
        min_size: int - Minimum component size for felzenszwalb segmentation.
    Returns
    -------
        img : ndarray
            image with region label
            region label is stored in the 4th value of each pixel [r,g,b,(region)]
        regions : array of dict
            [
                {
                    'rect': (left, top, right, bottom),
                    'labels': [...]
                },
                ...
            ]
    '''
    assert im_orig.shape[2] == 3, "3ch image is expected"
    # load image and get smallest regions
    # region label is stored in the 4th value of each pixel [r,g,b,(region)]
    img = _generate_segments(im_orig, scale, sigma, min_size)
    if img is None:
        return None, {}
    imsize = img.shape[0] * img.shape[1]
    R = _extract_regions(img)
    # extract neighbouring information
    neighbours = _extract_neighbours(R)
    # calculate initial similarities
    S = {}
    for (ai, ar), (bi, br) in neighbours:
        S[(ai, bi)] = _calc_sim(ar, br, imsize)
    # hierarchal search
    while S != {}:
        # get highest similarity
        # i, j = sorted(S.items(), cmp=lambda a, b: cmp(a[1], b[1]))[-1][0]
        i, j = sorted(list(S.items()), key = lambda a: a[1])[-1][0]
        # merge corresponding regions
        t = max(R.keys()) + 1.0
        R[t] = _merge_regions(R[i], R[j])
        # mark similarities for regions to be removed
        key_to_delete = []
        for k, v in S.items():
            if (i in k) or (j in k):
                key_to_delete.append(k)
        # remove old similarities of related regions
        for k in key_to_delete:
            del S[k]
        # calculate similarity set with the new region
        for k in filter(lambda a: a != (i, j), key_to_delete):
            n = k[1] if k[0] in (i, j) else k[0]
            S[(t, n)] = _calc_sim(R[t], R[n], imsize)
    # ... 
    regions = []
    for k, r in R.items():
        regions.append({
            'rect': (
                r['min_x'], r['min_y'],
                r['max_x'] - r['min_x'], r['max_y'] - r['min_y']),
            'size': r['size'],
            'labels': r['labels']
        })
    return img, regions
```

## Reference




> 1 当使用inline数学公式且公式经过GFM排版之后都在同一行 使用`$...$`符号<br>
> 2 当希望数学公式单独成行或者经过GFM排版之后占用多行 应当使用`$$...$$`符号<br>
> 3 对于表示条件概率 需要表示竖线的时候`|` 应当使用`\mid` 而不是直接在键盘上打出`|` => 容易被编辑器认为是一个md制表符<br>
> 4 在md引入图片的时候 不要使用`<center>`和`</center>` 在这篇文档的编辑过程中vscode的preview插件在使用了上述符号之后 导致下一段的数学公式预览显示不正常<br>
> 5 使用md的时候 单独的两段文字上下需要空出一行<br>
> 6 想要强制换行的时候 需要使用`<br>`而不是`<enter>`<br>
> 7 特殊字符如果想要避免和md解析关键字冲突 应当使用``将关键字包含在内
