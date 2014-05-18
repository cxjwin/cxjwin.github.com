---
layout: post
title: "使用UDP与模拟器App交互"
description: ""
categories:iOS
tags: [Test]
---
&emsp;&emsp;我们知道PC上的程序可以监听终端的输入输出,但是模拟器的终端我一直没找到,如果有找到的可以告诉鄙人一下.  
&emsp;&emsp;一般的如果我们想需要一个方法在稍后特定的时刻触发,那么用的最多的就是延时方法.但是我们想对该方法进行手动的控制的话呢?我第一个想到的便是监听标准输入,获取输入文字以后将文字转换成对应的命令.但是我试了一下,iOS模拟器上的标准输入不知道是什么,google一翻后也没查到个所以然.后来灵机一动,觉得我用UDP给模拟器发消息不是也是一样的效果么(就像聊天客户端一样).  
&emsp;&emsp;不想自己手动再实现一遍Socket编程,所以就去github找了一下,CocoaAsyncSocket这个就不错,而且模块是可以独立使用的,我使用UDP的话,就导入GCDAsyncUdpSocket.h/GCDAsyncUdpSocket.m这两个源文件就可以了.然后将UDP消息通过通知发出去,对应的类接受到消息后,转化成@selector,并且执行.  
&emsp;&emsp;这里我做了一个Demo,同时结合对于数组的KVO,来模拟数据增加或是减少后reload UITableView的情况.  
&emsp;&emsp;运行Demo,打开终端输入 cat > /dev/udp/127.0.0.1/8080, 然后输入 addObjects 试一下.  
&emsp;&emsp;[Demo地址](https://github.com/cxjwin/Demo_Input.git)