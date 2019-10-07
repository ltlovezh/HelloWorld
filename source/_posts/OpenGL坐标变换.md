---
title: OpenGL坐标变换
date: 2017-10-15 15:54:49
tags:
 - OpenGL
categories:
 - OpenGL
mathjax: true
---
## 基础概述
众所周知，OpenGL是一个3D图形库，在终端设备上广泛使用。但是我们的显示设备都是2D平面，那么OpenGL怎么把3D图形映射到2D屏幕那？这就是OpenGL坐标变换所要完成的工作。
一般情况下，我们总是通过一个2D屏幕，观察3D世界。因此，我们实际看到的是3D世界在2D屏幕上的一个投影。通过OpenGL坐标变换，我们可以在一个给定的观察视角下，把3D物体投影到2D屏幕上，再经过后面的光栅化和片元着色，整个3D物体就映射成了2D屏幕上的像素。
OpenGL的坐标变换流程如下所示：
![OpenGL坐标变换过程](http://7xs2qy.com1.z0.glb.clouddn.com/OpenGL%E5%9D%90%E6%A0%87%E5%8F%98%E6%8D%A2%E6%B5%81%E7%A8%8B.jpg)

<!-- more -->

---

由于Hexo MarkDown不能很好的支持各种数学公式，所以需要跳转到[OpenGL坐标变换](https://www.zybuluo.com/ltlovezh/note/911669)查看。
