---
layout: post
title: "iOS 下载图片前如何预取图片的大小"
description: ""
categories: iOS
tags: [Image]
---

&emsp;&emsp;最近练习做一个微博的项目，看到新浪微博的图片其实是可以根据图片的大小进行预览区域大小的设置，如果固定区域大小有时候会导致图片变形比较难看。google了很久，一直没有找到答案，如果是打图片的大小单独对应一组数据然后放在微博的json数据中返回过来，那么也好解决，但是微博并没有提供这样的接口。后来我又想是否有这样的请求命令可以直接索取图片的大小，那样的话我也可以不用加载完图片才能知道图片的大小。可惜也没找到这样的命令。

&emsp;&emsp;后来我觉得从最原始的方式开始探索，我觉得图片就是文件，文件就有文件头，一般的文件头里面都会有文件的一些常规信息，可能也包括图片的大小。所以，我在数据请求的时候，第一次请求文件头的数据或是更精确的话得到图片大小所对应的字段的数据，那么整个包可能只需要很少的字节就能得到图片的大小，有了图片得大小，我们就能设置预览区域的大小。

&emsp;&emsp;但是，还有一个问题，图片有很多种格式，所以文件头肯定是不一样的，没办法这里我是根据URL请求的后缀名进行区分的。后续我测试了下，微博图片主要就jpg和gif两种格式，然后我再加上常用的png格式，这个demo中我就是分析典型的三种图片格式。

&emsp;&emsp;首先，对于这三种格式，大家可以百度下，会有比较详细的格式介绍，当然很多内容我们并不需要，这里我们只要知道图片大小所在的据数段的位置即可。

&emsp;&emsp;具体格式的信息我这边就不描述了，百度就很容易的查到。

&emsp;&emsp;当然jpg格式有点复杂，因为我在测试的时候，图片大小所在的字段位置是不固定的，所以会麻烦些，具体见Demo中的分析。

png的分析，png格式图片大小的字段是在16-23，所以请求的时候只需要请求8字节即可（是不是很小）

```objective_c
- (void)downloadPngImage
{
  NSString *URLString = @"http://img2.3lian.com/img2007/13/29/20080409094710646.png";
  NSMutableURLRequest *request = [[NSMutableURLRequest alloc] initWithURL:[NSURL URLWithString:URLString]];
  [request setValue:@"bytes=16-23" forHTTPHeaderField:@"Range"];
  [[NSURLConnection connectionWithRequest:request delegate:self] start];
}
```

对应的每一位都需要进行转化，就能得到具体的数值

```objective_c
- (CGSize)pngImageSizeWithHeaderData:(NSData *)data
{
  int w1 = 0, w2 = 0, w3 = 0, w4 = 0;
  [data getBytes:&w1 range:NSMakeRange(0, 1)];
  [data getBytes:&w2 range:NSMakeRange(1, 1)];
  [data getBytes:&w3 range:NSMakeRange(2, 1)];
  [data getBytes:&w4 range:NSMakeRange(3, 1)];
  int w = (w1 << 24) + (w2 << 16) + (w3 << 8) + w4;
  int h1 = 0, h2 = 0, h3 = 0, h4 = 0;
  [data getBytes:&h1 range:NSMakeRange(4, 1)];
  [data getBytes:&h2 range:NSMakeRange(5, 1)];
  [data getBytes:&h3 range:NSMakeRange(6, 1)];
  [data getBytes:&h4 range:NSMakeRange(7, 1)];
  int h = (h1 << 24) + (h2 << 16) + (h3 << 8) + h4;

  return CGSizeMake(w, h);
}
```

jpg格式比较复杂所以先得了解清楚具体个字段的意思

因为图片大小所在的字段区域不确定，所以我们要扩大请求范围

这里209字节里面应该就已经包含全了所有的数据（这里我查了一些资料，也看了几个不同jpg的文件头16进制信息）

不一定就完全正确，但是分析微博的jpg图片大小暂时没有什么问题

```objective_c
- (void)downloadJpgImage
{
  NSString *URLString = @"http://ww3.sinaimg.cn/thumbnail/673c0421jw1e9a6au7h5kj218g0rsn23.jpg";
  NSMutableURLRequest *request = [[NSMutableURLRequest alloc] initWithURL:[NSURL URLWithString:URLString]];
  [request setValue:@"bytes=0-209" forHTTPHeaderField:@"Range"];
  [[NSURLConnection connectionWithRequest:request delegate:self] start];
}

- (CGSize)jpgImageSizeWithHeaderData:(NSData *)data
{
  if ([data length] <= 0x58) {
      return CGSizeZero;
  }

  if ([data length] < 210) {// 肯定只有一个DQT字段
    short w1 = 0, w2 = 0;
    [data getBytes:&w1 range:NSMakeRange(0x60, 0x1)];
    [data getBytes:&w2 range:NSMakeRange(0x61, 0x1)];
    short w = (w1 << 8) + w2;
    short h1 = 0, h2 = 0;

    [data getBytes:&h1 range:NSMakeRange(0x5e, 0x1)];
    [data getBytes:&h2 range:NSMakeRange(0x5f, 0x1)];
    short h = (h1 << 8) + h2;
    return CGSizeMake(w, h);
  } else {
    short word = 0x0;
    [data getBytes:&word range:NSMakeRange(0x15, 0x1)];
    if (word == 0xdb) {
      [data getBytes:&word range:NSMakeRange(0x5a, 0x1)];
      if (word == 0xdb) {// 两个DQT字段
        short w1 = 0, w2 = 0;
        [data getBytes:&w1 range:NSMakeRange(0xa5, 0x1)];
        [data getBytes:&w2 range:NSMakeRange(0xa6, 0x1)];
        short w = (w1 << 8) + w2;

        short h1 = 0, h2 = 0;
        [data getBytes:&h1 range:NSMakeRange(0xa3, 0x1)];
        [data getBytes:&h2 range:NSMakeRange(0xa4, 0x1)];
        short h = (h1 << 8) + h2;
        return CGSizeMake(w, h);
      } else {// 一个DQT字段
        short w1 = 0, w2 = 0;
        [data getBytes:&w1 range:NSMakeRange(0x60, 0x1)];
        [data getBytes:&w2 range:NSMakeRange(0x61, 0x1)];
        short w = (w1 << 8) + w2;
        short h1 = 0, h2 = 0;

        [data getBytes:&h1 range:NSMakeRange(0x5e, 0x1)];
        [data getBytes:&h2 range:NSMakeRange(0x5f, 0x1)];
        short h = (h1 << 8) + h2;
        return CGSizeMake(w, h);
      }
    } else {
      return CGSizeZero;
    }
  }
}
```

gif的分析和png差不多，不过这里得到得应该是第一张图片的大小  

```objective_c
- (void)downloadGifImage
{
  NSString *URLString = @"http://img4.21tx.com/2009/1116/92/20392.gif";
  NSMutableURLRequest *request = [[NSMutableURLRequest alloc] initWithURL:[NSURL URLWithString:URLString]];
  [request setValue:@"bytes=6-9" forHTTPHeaderField:@"Range"];
  [[NSURLConnection connectionWithRequest:request delegate:self] start];
}

- (CGSize)gifImageSizeWithHeaderData:(NSData *)data
{
  short w1 = 0, w2 = 0;
  [data getBytes:&w1 range:NSMakeRange(0, 1)];
  [data getBytes:&w2 range:NSMakeRange(1, 1)];
  short w = w1 + (w2 << 8);
  
  short h1 = 0, h2 = 0;
  [data getBytes:&h1 range:NSMakeRange(2, 1)];
  [data getBytes:&h2 range:NSMakeRange(3, 1)];
  short h = h1 + (h2 << 8);
  return CGSizeMake(w, h);
}
```

[Github Demo](https://github.com/cxjwin/TEST_ImageURL.git)
