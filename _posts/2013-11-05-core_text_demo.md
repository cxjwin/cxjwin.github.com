---
layout: post
title: "CoreText Demo系列"
description: "CoreText & TextKit Demo"
categories: iOS
tags: [CoreText]
---

&emsp;&emsp;承接之前CocoaChina上的[帖子](http://www.cocoachina.com/bbs/read.php?tid=110499)。那时候用的还是第三方的OHAttributedLabel，在此基础上添加一些功能。后来在项目中也用到了OHAttributedLabel发现有个问题，因为Label本身显示的字数超过2W(大概数字)的时候显示是空白界面，文字巨多的时候还是用相应的重用规则会比较好，当然UITextView就不会存在这个，这时候另一个第三方框架DTCoreText就更适合了，另外DTCoreText貌似只支持ARC，这两个框架GitHub上也一直有更新，是比较优秀的CoreText第三方库。当然也可以用UITableView来简便的实现重用，性能也不错，公司的项目上我就是用的这个方法，几万字在3GS上也跑的呼呼的。  
&emsp;&emsp;这里先介绍一篇很好的中文博客，我的很多基础知识和Demo上的功能是借鉴的上面的，上面的Demo也很不错。  
&emsp;&emsp;然后不准备讲太多理论性的东西，怕讲不好。因为CoreText一直断断续续都有学习，没一次也都会发现之前很多理解是错误的，然后也会有一些新的理解。所以这里我主要放我自己写的两个Demo，供大家学习交流。  

&emsp;&emsp;下面是几个个人认为Demo中比较重要的方法  

* CTFrameGetLineOrigins --> 这个方法是获取每一行的原点，这个在计算字体的坐标的时候会用到。

* CTLineGetGlyphRuns --> 这个方法就是获取一行里面所有的CTRun了，因为CTLine是由一个个CTRun组合而成的。

* CTRunGetAttributes --> 这个是获取CTRun的一些属性，这个为什么的重要？这个方法返回的是一个字典，当然字典里面除了一些系统属性外，你之前设置的一些自定义属性也能获取到，正是这样我们可以通过自定义的属性来特殊处理个别的CTRun（比如图片，链接等）。

* CTLineGetOffsetForStringIndex -->这个方法是获取具体的文字距离这行原点的距离，当然也是算尺寸用的。

* CTLineGetStringIndexForPosition -->这个方法是获取点击时我们点击的是哪一个文字，是给点击链接时显示点击效果用的。因为链接比较长，可能会超过一行，我们需要根据点击的文字找出指定的链接，然后在整个链接区域绘制背景色，这个是比较复杂的部分。

* 绘制方式有CTFrameDraw，CTLineDraw，CTRunDraw，Demo用的是CTRunDraw灵活性更强。

>PS:CoreText有一个不算bug的bug，因为之前都是用空格字符区域来显示表情，这带来一个问题，当连续的空格很多时，CoreText时会将他们绘制成一行的，是不会换行的，这个我在stackoverflow的时候，也没有找到很好的办法。现在我用“－”取代空格，但是又有另一个问题，“－”字会出现图片区域上面，那这时候需要调整下绘制顺序，所以Demo的绘制顺序是 --->  
1：链接的背景，2：文本, 3：图片。绘制图片的时候就不需要绘制文字了(不使用CTRunDraw方法)。绘制文字之前先绘制背景（不然背景会盖住文字）。

* 以下是Demo的github链接
Demo中还用到SDK自带的正则方法，个人感觉系统的还是挺好用的。另Demo中按个人理解做了一定注释，当然错误肯定是难免的，欢迎指正。    
[https://github.com/cxjwin/CoreTextDemo_cxjwin](https://github.com/cxjwin/CoreTextDemo_cxjwin)

* iOS7有了TextKit 使用起来更为方便，功能更为强大，所以我用TextKit做了一份相同功能的Demo  
[https://github.com/cxjwin/CoreTextDemo_iOS7.git](https://github.com/cxjwin/CoreTextDemo_iOS7.git)

