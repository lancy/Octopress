---
title: 使用CoreLocation来跟踪用户距离
author: lancy
layout: post
comments: true
categories:
  - iOS
tags:
  - cocoa
  - CoreLocation
  - GPS
  - iOS
  - LocationManager
  - OBJC
  - Programming
---
# 使用CoreLocation来跟踪用户距离

## 背景
CoreLocation是一个强大的Framework，他能帮助开发使其免于复杂的位置处理而专注于应用逻辑的开发。然而CoreLocation并没有提供的对用户移动距离的检测，当我们开发跑步类运动类应用时，就不可避免的需要这项功能。凑巧有一个朋友让我帮忙做一个GPS模块，故而就有了CYLocationManager。

代码在Github开源托管，[传送门](https://github.com/lancy/LocationManager)

## 实现说明

Readme有详细的使用说明，我在这里主要描述一下实现的一些要点。

基本的思路既是不断的采样用户数据，过滤掉误差较大的数据，取相对误差较小的数据进行记录，然后计算相邻记录点之间的距离。

简单描述一下几个要点：

1. 当用户开始运动，程序开始追踪，设置一个强制标记，（needForceCalculation），表示程序应该忽略其他因素，立刻获取一个点坐标。用做起始值。
2. 设置了CLLocationManager.headingFilter，使得程序能在用户转向的时候收到通知，此时设置一个强制标记（needForceCalculation），使得程序在用户转向的时候，记录下转向时所在的位置，以减少误差。
3. 设置CLLocationManager.distanceFilter，使得程序在变化的位置大于一定数值时该更新位置才算为有效，可以避免用户在一个地方停留，由于误差记录距离依然增长。
4. 当程序获得位置更新时，若精度合格，切时间戳合理，则加入一个数组，用于之后的计算。若精度大于某个阀值，则认为该位置对跟踪距离无帮助，此时将该位置舍去。
5. 数组currentKeepLocations来记录最近更新的k个位置，并每隔t秒，从该数组中，取出精度最高的位置记录。（精度见CLLocation.horizontalAccuracy）
6. 注意，当用户停止运动时，位置将无法得到更新，此时需要设置一个timer，令其在一定时间内强制获得一个位置。
7. 该程序还可以通过每次更新位置时获得的位置的精确度来判断GPS信号的强弱。

## 联系我
如果你对这个程序有疑问，请联系我
