---
layout: post
title: "利用runtime隐藏QLPreviewController的分享按钮"
description: ""
categories: iOS
tags: [Runtime]
---

&emsp;&emsp;最近公司做一个项目，会用自定预览界面，首先想到的就是QLPreviewController，因为这是Apple自带的框架，效率上也比WebView好些。但是QLPreviewController的自定义特性比较差，我们的界面不需要QLPreviewController 的 NavigationBar 右上角的 分享按钮，纠结了很久一直没有找到比较好的解决方法，后来还是万能的github给出了一个较为理想的答案。  

&emsp;&emsp;老外给的建议是用runtime改写原来的方法，用的是UINavigationItem的类别。如果是QLPreviewController类那么就自定义的setRightBarButtonItem或者setRightBarButtonItems方法，其他的类就按照原来的方法执行。乍一看其实挺完美的一个方案，但是还是有隐患的，因为是类别方法，而且load的时候就加载了，那么后续如果其他人不知道有这种情况，想用系统的分享按钮也用不着了，因为所有的QLPreviewController都会用到这个类别。  

### 方案一

```objective_c
+ (void)load
{
  MethodSwizzle(self, @selector(setRightBarButtonItem:animated:), @selector(override_setRightBarButtonItem:animated:));
  MethodSwizzle(self, @selector(setRightBarButtonItems:animated:), @selector(override_setRightBarButtonItems:animated:));
}

void MethodSwizzle(Class class, SEL originalSEL, SEL overrideSEL) 
{
  Method originalMethod = class_getInstanceMethod(class, originalSEL);
  Method overrideMethod = class_getInstanceMethod(class, overrideSEL);

  if (class_addMethod(class, originalSEL, method_getImplementation(overrideMethod), method_getTypeEncoding(overrideMethod))) {
    class_replaceMethod(class, overrideSEL, method_getImplementation(originalMethod), method_getTypeEncoding(originalMethod));
  } else {
    method_exchangeImplementations(originalMethod, overrideMethod);
  }
}

- (void)override_setRightBarButtonItem:(UIBarButtonItem *)item animated:(BOOL)animated
{
  if (item && [item.target isKindOfClass:[QLPreviewController class]] && item.action == @selector(actionButtonTapped:)){
    QLPreviewController *previewController = (QLPreviewController *)item.target;
    [self override_setRightBarButtonItem:previewController.navigationItem.rightBarButtonItem animated:animated];
  } else {
    [self override_setRightBarButtonItem:item animated:animated];
  }
}

- (void)override_setRightBarButtonItems:(NSArray *)items animated:(BOOL)animated
{
  if ([items count] == 0) {
    return;
  } else {
    UIBarButtonItem *firstItem = [items objectAtIndex:0];
    BOOL override = NO;
    if (firstItem && [firstItem.target isKindOfClass:[QLPreviewController class]] && firstItem.action == @selector(actionButtonTapped:)) {
      override = YES;
    }
    if (override) {
      QLPreviewController *previewController = (QLPreviewController *)firstItem.target;
      [self override_setRightBarButtonItems:previewController.navigationItem.rightBarButtonItems animated:animated];
    } else {
      [self override_setRightBarButtonItems:items animated:animated];
    }
  }
}
```

*1.因为load方法是UIBarButtonItem类加载的时候就会调用的，所以保证了有效性  
2.然后交换原来的方法，用我们自己的方法实现  
3.这里作者用了个技巧，因为已经交换了方法，所以再调用一次override_\*方法相当于调用了原来的方法*  

### 方案二

&emsp;&emsp;既然这样我做一下小改动，自定义一个QLPreviewController的子类，再界面显示前我替换掉原来的方法，界面显示完以后我就把原来的方法再交换回来，那么其他的类包括QLPreviewController本身因该是不会收到影响的。  

```objective_c
+ (void)exchangeMethod
{
  MethodSwizzle(self, @selector(setRightBarButtonItem:animated:), @selector(override_setRightBarButtonItem:animated:));
  MethodSwizzle(self, @selector(setRightBarButtonItems:animated:), @selector(override_setRightBarButtonItems:animated:));
}

+ (void)noExchangeMethod
{
  MethodSwizzle(self, @selector(override_setRightBarButtonItem:animated:), @selector(setRightBarButtonItem:animated:));
  MethodSwizzle(self, @selector(override_setRightBarButtonItems:animated:), @selector(setRightBarButtonItems:animated:));
}
```

### 方案三
  
&emsp;&emsp;那还有没有其他方案？这里我稍加修改了下，我不用类别，直接使用runtime行不行？因为只是隐藏rightBarButtonItem或是rightBarButtonItems那么我简单的执行一个空方法就可以了，末了我在把原来的方法还原回去。  

首先定义两个函数指针变量存储原来的实现方法  

```objective_c
@implementation CSPreviewController
{
  IMP origImpOne;
  IMP origImpTwo;
}
```

这里是两个空方法，注意格式  

```objective_c
static void override_setRightBarButtonItem(id _self, SEL __cmd, UIBarButtonItem *item, BOOL animated)
{
  //
}

static void override_setRightBarButtonItems(id _self, SEL __cmd, NSArray *items, BOOL animated)
{
  //
}
```

runtime实现  

```objective_c
- (void)overrideMethod
{
  SEL selectorOneToOverride = @selector(setRightBarButtonItem:animated:);
  Method methodOne = class_getInstanceMethod([UINavigationItem class], selectorOneToOverride);
  origImpOne = class_getMethodImplementation([UINavigationItem class], selectorOneToOverride);
  method_setImplementation(methodOne, (IMP)override_setRightBarButtonItem);

  SEL selectorTwoToOverride = @selector(setRightBarButtonItems:animated:);
  Method methodTwo = class_getInstanceMethod([UINavigationItem class], selectorTwoToOverride);
  origImpTwo = class_getMethodImplementation([UINavigationItem class], selectorTwoToOverride);
  method_setImplementation(methodTwo, (IMP)override_setRightBarButtonItems);
}

- (void)noOverrideMethod
{
  SEL selectorOneToOverride = @selector(setRightBarButtonItem:animated:);
  Method methodOne = class_getInstanceMethod([UINavigationItem class], selectorOneToOverride);
  method_setImplementation(methodOne, origImpOne);

  SEL selectorTwoToOverride = @selector(setRightBarButtonItems:animated:);
  Method methodTwo = class_getInstanceMethod([UINavigationItem class], selectorTwoToOverride);
  method_setImplementation(methodTwo, origImpTwo);
}
```

放在super方法之前调用，因为super调用的时候已经开始调用set方法了，再改就来不及了。  

```objective_c
- (void)viewWillAppear:(BOOL)animated
{
  [self overrideMethod];
  [super viewWillAppear:animated];
}

- (void)viewWillDisappear:(BOOL)animated
{
  [self noOverrideMethod];
  [super viewWillDisappear:animated];
}
```

如果需要更详细的自定义，那么修改以上方案即可。

**PS:[Demo源码](https://github.com/cxjwin/TEST_PreviewController.git)**
