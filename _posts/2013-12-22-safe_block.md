---
layout: post
title: "MRC下安全的Block"
description: ""
categories:
tags: [iOS]
---

&emsp;&emsp;iOS4以后引入的block，一个比较方便且实用的功能。但是自己在开发的时候遇到了不少的坑，很多都是和内存管理相关的，后来iOS5.0以后有了ARC，有了__weak关键字，所以block使用也就更安全了。  

&emsp;&emsp;但是，吐槽下我们公司。对于像我们公司这种，还在支持iOS4.3，还在用MRC的来说，很多后来方便的框架和工具都不能用了。而那个坑还是那个坑。 

&emsp;&emsp;最近在看C++的东西，无意中又想起了这个坑，所以立即写了个Demo，应该算是一种思路。  

&emsp;&emsp;MRC中使用block，为了避免在block中retain外部的变量导致引用计数的增加，我们往往要在block中使用的变量前加上 *__block* 关键字，这样就不会retain了，比如*__block typeof(self) weakSelf = self*;  

&emsp;&emsp;但是这里又会带来另外一个问题，因为block很多时候是用在异步的情况下，如果这时候别的地方已经把self给释放掉了，那么那个weakSelf就变成了野指针，这个时候再在block中去调用weakSelf很有可能就崩溃掉了。这个坑在网络调用的时候害了我很多次。  

&emsp;&emsp;我尝试了很多方法一直都没有很好的效果，比如用一个static变量存储self指针，因为static变量在block内是不会retain的，它在整个包内(当前.m文件中)是共用的。但是一个类共享一个static变量，那么我创建很多个实例对象的时候这个方法就失效了。  

&emsp;&emsp;后来我想起了一个类似引用计数的方法，创建一个实例就＋1，释放一个就－1，当总数 > 0 的时候才调用block。这样又有一个问题，就是假如是一个延时方法调用block，在这个延时的时间内我释放了所有的实例，然后又重新创建一遍，这个时候总数还是 > 0，block会执行下去，但是原先的实例已经释放掉了，所以还是会崩溃。  

&emsp;&emsp;再后来，我网上查了下4.3下*__weak*关键字替代方案，果真有，github上有个专门的类模拟*__weak*效果。具体叫什么我忘了。但是这里我没用，因为我觉得我只是用在block上，而不是替代*__weak*。所以还是想个简单点的方法。  

&emsp;&emsp;当然很多时候没必要想那么复杂。既然不想retain，那么我使用它的指针就行了，那么我这里想到的就是用一个全局的NSMutbleSet去存储所有存活的需要用在block中的对象的指针。 举个例子，用到BLObject对象的话， 我们init的时候 把self的指针add到Set中，dealloc的时候从Set中remove掉，判断指针是不是有效只要判断Set有没有这个指针就行了，这里我把指针变量转换成NSValue便于存储。 初步证明还是有效的。  

&emsp;&emsp;Demo如何证明？我们present到第二个viewController的时候，点击Test，这个时候会触发延时方法，如果在延时时间内（Demo中是2秒）你dismiss的话第二个viewController已经释放了，这个block执行的时候再调用外部的变量就有可能会崩溃掉。但是这里我们检查了原先的指针是不是有效，失效就立即返回，不会崩溃。  

[Demo](https://github.com/cxjwin/Safe_Block.git)