---
layout:     post
title:      "Image recognition in C using TensorFlow "
subtitle:   "使用TensorflowC做图像识别"
date:       2019-04-02 03:00:00 
author:     "Roche" 
header-img: "img/home-bg.jpg" 
tags:
    - Tensorflow 

---

## 前言

一直以来，我觉得如果可能，尽量把计算放到端设备上，这样会加速整个流程。  
这次是使用inception的模型做图像识别。

## 简介

简单来说，是图形的多分类。  
使用的是inception_v3_2016_08_28_frozen.pb这个模型。  
想要了解详情，左转搜索inception。

## 具体流程

简单说：

- 图形转化为输入格式
- 运行模型
- 获取运行结果

因为是图像识别，所以每一步都会麻烦一点。

## 简单的运行下

这里是输入，百度图片搜索的埃及猫

![cat](https://timgsa.baidu.com/timg?image&quality=80&size=b9999_10000&sec=1554144583744&di=2f02a300a91766776dfcce80b22cc5c7&imgtype=0&src=http%3A%2F%2Fs13.sinaimg.cn%2Forignal%2F4b9201031a43d400d03dc)

判定结果

![大概这样](https://roche-k.github.io/img/in-post/deep_learnig/2019-04-02-inception.png)

由于这个结果是从0开始的，实际上是287行

![大概这样](https://roche-k.github.io/img/in-post/deep_learnig/2019-04-02-inception_result.png)

结果是埃及猫，符合输入

## 总结

撸猫撸猫
