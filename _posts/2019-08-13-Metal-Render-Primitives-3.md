---
layout: post
title: Metal - Render Primitives（3）
date: 2019-08-13
Author: Xiaoye
tags: [Metal]
excerpt_separator: <!--more-->
toc: true
---


上一篇介绍了如何通过 `MTKView` 和 `render pass` 修改 view 的 contents。这一篇介绍如何通过渲染管线（`render pipeline`）绘制一个三角形。

 <!--more-->
 
这里渲染管线可以说是整个 `render pass` 的一部分，我们回忆一下第一篇文章介绍的渲染管线是什么

> 渲染管线，实际上指的是一堆原始图形数据途经一个输送管道，期间经过各种变化处理最终出现在屏幕的过程。通常情况下，渲染管线有三个阶段，其中光栅化阶段不可编程，其他两个可以
> 1. 顶点着色器（vertex stage），接收一组顶点数据数组，来限定显示／处理区域
> 2. 光栅化阶段（rasterization stage），在光栅化阶段，确定哪些像素位于边界，裁剪超出边界的像素
> 3. 片段着色器（fragment stage），计算每一个像素最终的颜色值



我们再回忆一下第二篇文章中的 `commands` 、`textures` 的说明，渲染管线可以更具体的理解成这样的一个过程

> 处理所有的所有的绘制相关的 commands，并且将处理后的数据保存在 textures 的过程



对渲染管线有所了解后下面通过代码看看是如何运作的



