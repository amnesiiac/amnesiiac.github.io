---
layout: post
title: "Non-maximum-suppresion methods"
subtitle: '用于目标检测的NMS算法介绍|代码实现'
author: "twistfatezz"
header-style: text
# header-img: "img/post-bg-alitrip.jpg"
# header-mask: 0.1
mathjax: true
date: 2019-10-01 20:12 
lang: ch 
catalog: true
categories: documentation 
tags:
  - nn toolkit
  - Time 2019
---

## Introduction NMS
一张图片理解NMS的基本功能：

<center><img src="/img/in-post/nms/nms.png" width="80%"></center>

## Code - NMS
```c++
// use bubblesort to sort indices according to x(confidence score in each bbox)
// can choose quicksort or other method for better runtime speed
static void sort(int n, const float* x, int* indices){  
    int i, j;  
    for(i=0; i<n; i++){
        for(j=i+1; j<n; j++){  
            if(x[indices[j]] > x[indices[i]]){  
                int index_tmp = indices[i];  
                indices[i] = indices[j];  
                indices[j] = index_tmp;  
            } 
        }
    }
}
```
```c++
// nms algorithm core
int nonMaximumSuppression(int numBoxes, const CvPoint *points, const CvPoint *oppositePoints, 
                          const float *score, float overlapThreshold, int *numBoxesOut, CvPoint **pointsOut, 
                          CvPoint **oppositePointsOut, float **scoreOut){  
  
    // numBoxes:num-of-bbox  points:left-point  oppositePoints:right-point 
    // score:confidence   overlapThreshold:iou-thresh   numBoxesOut: num-of-output-bbox
    // pointsOut:left-point-output oppositePoints:right-point-output
    // scoreOut:score-of-output-bbox
    int i, j, index;  
    float* box_area = (float*)malloc(numBoxes * sizeof(float));
    int* indices = (int*)malloc(numBoxes * sizeof(int));
    int* is_suppressed = (int*)malloc(numBoxes * sizeof(int));
    // 初始化indices、is_supperssed、box_area信息   
    for(i = 0; i < numBoxes; i++)  
    {  
        indices[i] = i;  
        is_suppressed[i] = 0;  
        box_area[i]=(float)((oppositePoints[i].x - points[i].x+1) *  
                                (oppositePoints[i].y - points[i].y+1));  
    }  
    // sort    
    sort(numBoxes, score, indices);  
    for(i=0; i<numBoxes; i++){// choose bbox with maximum confidence each epoch
        if(!is_suppressed[indices[i]]){// the chosen bbox should not be suppressed
            for(j=i+1; j<numBoxes; j++){// for the left bboxes
                if(!is_suppressed[indices[j]]){// the chosen bbox for comparison should not be suppressed
                    int x1max = max(points[indices[i]].x, points[indices[j]].x);// left x   
                    int x2min = min(oppositePoints[indices[i]].x, oppositePoints[indices[j]].x);// right x   
                    int y1max = min(points[indices[i]].y, points[indices[j]].y);// left y   
                    int y2min = max(oppositePoints[indices[i]].y, oppositePoints[indices[j]].y);// right y
                    int overlapWidth = x2min-x1max+1;// width
                    int overlapHeight = y2min-y1max+1;// height   
                    if(overlapWidth>0 && overlapHeight>0){  
                        float overlapPart = (overlapWidth*overlapHeight)/box_area[indices[j]];// iou
                        if (overlapPart > overlapThreshold){  
                            is_suppressed[indices[j]]=1;// set as suppressed   
                        }  
                    }  
                }  
            }  
        }  
    }  
    // count final output num of bboxes
    *numBoxesOut=0;
    for(i=0; i<numBoxes; i++){  
        if(!is_suppressed[i]){(*numBoxesOut)++;}
    }  
    *pointsOut = (CvPoint *)malloc((*numBoxesOut) * sizeof(CvPoint));// left out
    *oppositePointsOut = (CvPoint *)malloc((*numBoxesOut) * sizeof(CvPoint));// right out
    *scoreOut = (float *)malloc((*numBoxesOut) * sizeof(float));// iou score
    index = 0;  
    for(i=0; i<numBoxes; i++){  
        if(!is_suppressed[indices[i]]){// save unsuppressed bbox into output vec
            (*pointsOut)[index].x = points[indices[i]].x;  
            (*pointsOut)[index].y = points[indices[i]].y;  
            (*oppositePointsOut)[index].x = oppositePoints[indices[i]].x;  
            (*oppositePointsOut)[index].y = oppositePoints[indices[i]].y;  
            (*scoreOut)[index] = score[indices[i]];  
            index++;  
        }
    }  
    // free mem
    free(indices);   
    free(box_area);   
    free(is_suppressed);   
    return LATENT_SVM_OK;// set flag
}
```

## Introduction soft NMS
上面的普通的NMS方法有个bug，它不能处理下面图片中的情况：

<center><img src="/img/in-post/nms/softnms0.png" width="80%"></center>

图中两匹马靠的很近，而且它们的confidence分别为0.95和0.8。由于相距很近，它们之间的IOU超过了NMS的的阈值，如果使用上面的普通NMS算法，则confidence较小的那个bbox(green bbox)将会被抑制，这显然是不正确的。为了能够弥补这个bug，显然应该将green bbox中的confidence降低而不是将它进行抑制(discard)。具体地，和当前轮次confidence最大的bbox计算IOU，如果IOU超过阈值，那么对于它进行衰减，而不是直接作为suppression丢弃。进一步地，如果IOU重叠数值越大，说明它越可能和当前轮次confidence最大的bbox指向同一个ground_truth物体,因此, iou数值越大，confidence score衰减的越大，通过设置合理的衰减幅度，可以实现更好的"suppression"效果。

**对于普通的NMS算法：**
$$
s_{i}=\left\{\begin{array}{ll}{s_{i},} & {\operatorname{iou}\left(\mathcal{M}, b_{i}\right)<N_{t}} \\ {0,} & {\operatorname{iou}\left(\mathcal{M}, b_{i}\right) \geq N_{t}}\end{array}\right.
$$

**对于softNMS中的linear方式：**
$$
s_{i}=\left\{\begin{array}{ll}{s_{i},} & {\operatorname{iou}\left(\mathcal{M}, b_{i}\right)<N_{t}} \\ {s_{i}\left(1-\operatorname{iou}\left(\mathcal{M}, b_{i}\right)\right),} & {\operatorname{iou}\left(\mathcal{M}, b_{i}\right) \geq N_{t}}\end{array}\right.
$$

**对于softNMS中的gaussian方式：**
$$
s_{i}=s_{i} e^{-\frac{\operatorname{iou}\left(\mathcal{M}, b_{i}\right)^{2}}{\sigma}}, \forall b_{i} \notin \mathcal{D}
$$

其中，$\mathcal{M}$是每一轮中confidence最大的benchmark bbox，$b_{i}$是每一轮中用于和benchmark bbox计算iou的comparison bbox，$\mathcal{N}_t$表示用于决定是否suppress的阈值，$\mathcal{D}$表示每轮次中，"通过"suppression检验的作为descision bbox的集合，每一轮次都增加一个。$s_i$表示第$i$个bbox的confidence score。详细情况见下面的代码。

**补充关于softnms算法的使用经验**使用soft-nms需要比普通的nms算法计算量大，消耗更多的时间。而且不是所有的任务使用soft-nms算法都比nms算法好用，可以先对数据集中的bbox做个聚类分析，分析一下bbox位置先验都在什么地方。对于bbox重合度很高，物体之间距离很近很近的情况，使用soft-nms的确会有提升；但是对于bbox比较分散的情况，使用soft-nms有可能使得bbox处理结果变糟。

## Code - soft NMS
soft NMS的官方cython代码如下，代码来自：https://github.com/bharatsingh430/soft-nms/tree/master/lib/nms 中的cpu_nms.pyx
```python
import numpy as np
cimport numpy as np

cdef inline np.float32_t max(np.float32_t a, np.float32_t b):
    return a if a >= b else b

cdef inline np.float32_t min(np.float32_t a, np.float32_t b):
    return a if a <= b else b

def cpu_soft_nms(np.ndarray[float, ndim=2] boxes, float sigma=0.5, float Nt=0.3, 
                float threshold=0.001, unsigned int method=0):
    '''
    boxes (N,5) -> (x1,y1,x2,y2,confidence) | sigma is for guassian | Nt = threshold
    '''
    cdef unsigned int N = boxes.shape[0]
    cdef float iw, ih, box_area
    cdef float ua
    cdef int pos = 0
    cdef float maxscore = 0
    cdef int maxpos = 0
    cdef float x1,x2,y1,y2,tx1,tx2,ty1,ty2,ts,area,weight,ov

    for i in range(N):
        maxscore = boxes[i, 4]
        maxpos = i

        tx1 = boxes[i,0]
        ty1 = boxes[i,1]
        tx2 = boxes[i,2]
        ty2 = boxes[i,3]
        ts = boxes[i,4]
        pos = i + 1

	    # get max box
        while pos < N:
            if maxscore < boxes[pos, 4]:
                maxscore = boxes[pos, 4]
                maxpos = pos
            pos = pos + 1

	    # add max box as a detection 
        boxes[i,0] = boxes[maxpos,0]
        boxes[i,1] = boxes[maxpos,1]
        boxes[i,2] = boxes[maxpos,2]
        boxes[i,3] = boxes[maxpos,3]
        boxes[i,4] = boxes[maxpos,4]

	    # swap ith box with position of max box
        boxes[maxpos,0] = tx1
        boxes[maxpos,1] = ty1
        boxes[maxpos,2] = tx2
        boxes[maxpos,3] = ty2
        boxes[maxpos,4] = ts
        tx1 = boxes[i,0]
        ty1 = boxes[i,1]
        tx2 = boxes[i,2]
        ty2 = boxes[i,3]
        ts = boxes[i,4]
        pos = i + 1

	    # NMS iterations, note that N changes if detection boxes fall below threshold
        while pos < N:
            # benchmark bbox coordinate & confidence 
            x1 = boxes[pos, 0]
            y1 = boxes[pos, 1]
            x2 = boxes[pos, 2]
            y2 = boxes[pos, 3]
            s = boxes[pos, 4]

            area = (x2-x1+1) * (y2-y1+1)
            iw = (min(tx2, x2) - max(tx1, x1) + 1)
            if iw > 0:
                ih = (min(ty2, y2) - max(ty1, y1) + 1)
                if ih > 0:
                    # compute iou between benchmark bbox and comparison bbox
                    ua = float((tx2-tx1+1) * (ty2-ty1+1) + area - iw*ih)
                    ov = iw * ih / ua 
                    # linear
                    if method == 1: 
                        if ov > Nt: 
                            weight = 1 - ov
                        else:
                            weight = 1
                    # gaussian
                    elif method == 2: 
                        weight = np.exp(-(ov * ov)/sigma)
                    # original NMS
                    else: 
                        if ov > Nt: 
                            weight = 0
                        else:
                            weight = 1
                    # refresh comparison box confidence
                    boxes[pos, 4] = weight*boxes[pos, 4]
		    
                    # if box score falls below threshold, discard the box by swapping with last box, update N
                    if boxes[pos, 4] < threshold:
                        boxes[pos,0] = boxes[N-1, 0]
                        boxes[pos,1] = boxes[N-1, 1]
                        boxes[pos,2] = boxes[N-1, 2]
                        boxes[pos,3] = boxes[N-1, 3]
                        boxes[pos,4] = boxes[N-1, 4]
                        N = N - 1
                        pos = pos - 1
            pos = pos + 1
    keep = [i for i in range(N)]
    return keep
```

## Introduction softer NMS
回顾传统的nms算法的缺点：<br>
[1] 物体之间重叠太多，使用nms算法会将距离很近的其他物体的bbox删除。<br>
[2] 基于confidence进行，每个轮次只有最高confidence的被留下，然而有很多情况，iou准确度和confidence准确度不是强相关，因此选用最高的confidence的框也不一定那么精准。<br>
为了能够解决上述的问题，介绍softer nms。更多信息请参考论文[Softer-NMS: Rethinking Bounding Box Regression for Accurate Object Detection]。

## Preparations
softer nms主要有两个创新点：[1]: 提出了`KL-loss`:将定位置信度作为检测模型训练一部分;[2]: 提出了`softer-nms`:根据每个bbox的置信度来提高定位的精度。其中置信度越大，代表当前predict的bbox越能够贴近ground truth bbox。

**高斯函数对预测bbox建模:**使用高斯分布作为对于预测bbox的先验分布。分布可以写成：

$$
P_{\Theta}(x)=\frac{1}{2 \pi \sigma^{2}} e^{-\frac{\left(x-x_{e}\right)^{2}}{2 \sigma^{2}}}
$$

**$\delta$函数对于gt-bbox建模:**使用`dirac`分布函数作为gt-bbox的损失函数。dirac分布可以看成是高斯分布中$u=0$,且$\sigma$趋向于0的极限分布。softer nms对于gt-bbox的分布建模的具体形式如下：

$$
P_D(x)=\delta(x-x_g)
$$

关于上面两个公式的说明：对于预测bbox中的高斯函数，当$\sigma$越小，曲线越瘦高，当$\sigma$非常小的时候，变成了冲击函数(信号与系统)，同时$\sigma$越小，`gaussian`分布函数可以近似看成是一个`dirac`分布。也就是说，对于预测的bbox的分布无限接近于真实的分布。那么如何衡量两个部分之间的差异呢?很容易能够想到，使用KL散度作为loss即可。同时$\sigma$可以作为衡量预测的bbox有多大程度上接近真实的bbox，$\sigma$越小越接近，因此可以考虑使用$1-\sigma$作为bbox接近gt-bbox程度的指标。论文中改造的[fast-rcnn](/documentation/2019/10/06/post-fast-rcnn/)网络结构如下：

<center><img src="/img/in-post/nms/softernms.png" width="100%"></center>

公式中的$\sigma$就是图中的bbox_std，通过构建图中最下面的分支网络得到其数值，公式中的$x_e$通过图中第二个分支得到。接下来推导$L_{reg}$：

$$
L_{reg} = D_{KL}\left(P_{D}(x) \mid\mid P_{\Theta}(x)\right) = \int P_{D}(x) \log \frac{P_{D}(x)}{P_{\Theta}(x)} dx
$$

$$
=-\int P_{D}(x) \log P_{\Theta}(x) dx + \int P_{D}(x) \log P_{D}(x) dx
$$

$$
=-\int P_{D}(x) \log P_{\Theta}(x) dx + H\left(P_{D}(x)\right)
$$

$$
=-\log P_{\Theta}\left(x_{g}\right) + H\left(P_{D}(x)\right)
$$

将两个分布的公式代入上式得到：

$$
=\frac{\left(x_{g}-x_{e}\right)^{2}}{2 \sigma^{2}}+\frac{1}{2} \log \left(\sigma^{2}\right)+\frac{1}{2} \log (2 \pi)+H\left(P_{D}(x)\right)
$$

上面的式子中后面两项是常数项，因此，直接去掉即可，考虑前面两项，易知：

$$
D_{KL}\left(P_{D}(x) \mid\mid P_{\Theta}(x)\right) \propto \frac{\left(x_{g}-x_{e}\right)^{2}}{2 \sigma^{2}}+\frac{1}{2} \log \left(\sigma^{2}\right)
$$

对上面的式子求导，得到：

$$
\begin{aligned} \frac{d}{d x_{e}} D_{K L}\left(P_{D}(x) \| P_{\Theta}(x)\right) &=\frac{x_{e}-x_{g}}{\sigma^{2}} \\ \frac{d}{d \sigma} D_{K L}\left(P_{D}(x) \| P_{\Theta}(x)\right) &=-\frac{\left(x_{e}-x_{g}\right)^{2}}{\sigma^{-3}}-\frac{1}{\sigma} \end{aligned}
$$

发现，求导得到的结果中$\sigma$在分母上，优化$\sigma$的时候容易导致梯度爆炸，因此，令$\alpha=\frac{1}{\sigma^2}$，做一次参数转化，得到实际上我们$L_{reg}$的优化形式：

$$
D_{KL}\left(P_{D}(x) \mid\mid P_{\Theta}(x)\right) \propto \frac{\alpha}{2}\left(x_{g}-x_{e}\right)^{2}-\frac{1}{2} \log (\alpha+\epsilon) 
$$

$$
\frac{d}{d x_{e}} D_{K L}\left(P_{D}(x) \mid\mid P_{\Theta}(x)\right)=\alpha\left(x_{e}-x_{g}\right)
$$

$$
\frac{d}{d \alpha} D_{K L}\left(P_{D}(x) \mid\mid P_{\Theta}(x)\right)=\frac{\left(x_{e}-x_{g}\right)^{2}}{2}-\frac{1}{2(\alpha+\epsilon)}
$$

Softer-NMS的实现稍显复杂，需要网络重新训练以预测出四个坐标(x1, x2, y1, y2)的坐标置信度，然后用此置信度作为权值加权。实现了`多框合一`。

## Code - softer NMS
等待整理...

## Introduction adaptive NMS
关于adaptive NMS，详见论文[Adaptive NMS: Refining Pedestrian Detection in a Crowd]。待整理。
## Code - adaptive NMS
等待整理...

## Reference
> softnms offical realization: https://github.com/bharatsingh430/soft-nms/tree/master/lib/nms <br>
> "Improving Object Detection With One Line of Code" <br>
> http://blog.prince2015.club/2018/12/01/Soft-NMS/

> 1 当使用inline数学公式且公式经过GFM排版之后都在同一行 使用`$...$`符号<br>
> 2 当希望数学公式单独成行或者经过GFM排版之后占用多行 应当使用`$$...$$`符号<br>
> 3 对于表示条件概率 需要表示竖线的时候`|` 应当使用`\mid` 而不是直接在键盘上打出`|` => 容易被编辑器认为是一个md制表符<br>
> 4 在md引入图片的时候 不要使用`<center>`和`</center>` 在这篇文档的编辑过程中vscode的preview插件在使用了上述符号之后 导致下一段的数学公式预览显示不正常<br>
> 5 使用md的时候 单独的两段文字上下需要空出一行<br>
> 6 想要强制换行的时候 需要使用`<br>`而不是`<enter>`<br>
> 7 特殊字符如果想要避免和md解析关键字冲突 应当使用``将关键字包含在内