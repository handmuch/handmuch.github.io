---
layout:     post
title:      " 网易新闻下拉刷新效果实现 "
subtitle:   "Make a refresh effect like NetEase News"
date:       2015-10-28 12:00:00
author:     "PeterKwok"
header-img: "img/home-bg-o.jpg"
catalog: true
tags:
    - iOS开发
    - 交互动画
---

## 前言

**更换新blog的第一篇技术文章就这样开始了，PeterKwok的技术与生活将有更多东西和大家分享。**

***
	
这个文章的诞生是因为在工作中遇到了相应的需求，需要做一个可以自定义动画及刷新消息提醒的下拉刷新效果（如下图），产品经理的原话就是“我要一个想网易新闻的下拉刷新效果！”就是产品经理的这一句话，我就开始了对网上的下来刷新控件进行了一次大概的研究。

![Force Events](http://image18-c.poco.cn/mypoco/myphoto/20161020/15/17391699920161020153609021.gif?404x720_110)

## 正文

根据我百度和Github结合发现，iOS开发者用得比较多的下来刷新控件大概有这个三个，下面我们分析一下这个三个控件各自的特点。

#### EGORefreshTableHeaderView
基本上可以说是最早的一个下来刷新控件了，看Github上的提交差不多是6年前。

GOOD
 
1. EGORefreshTableHeaderView刷新动画及UI布局相对简单，可自定义及扩展性好，相对轻量，只有两添加一个类。

BAD

1. 全部采用delegate方式回调状态方法，UI布局及回调方法在同一个类，代码不够简洁，可读性较差
2. 需要使用scrollView delegate回调处理状态

#### MJRefresh
MJRefresh应该算是国内下拉刷新控件中的一个后起之秀了，作为一个国人写控件，有着庞大的用户群体，就证明了这个控件的质量。

GOOD

1. 使用方式简单，只需要小量代码就可以在源代码中集成下拉刷新
2. 使用了block作为方法回调，代码简约，可读性高
3. 刷新状态比较细分，定义清晰，对于刷新动画自定义有更好的支持，代码如下


``` objc 
/** 刷新控件的状态 */
typedef NS_ENUM(NSInteger, MJRefreshState) {
    /** 普通闲置状态 */
    MJRefreshStateIdle = 1,
    /** 松开就可以进行刷新的状态 */
    MJRefreshStatePulling,
    /** 正在刷新中的状态 */
    MJRefreshStateRefreshing,
    /** 即将刷新的状态 */
    MJRefreshStateWillRefresh,
    /** 所有数据加载完毕，没有更多的数据了 */
    MJRefreshStateNoMoreData
}; 
```

Bad

1. 自定义刷新动画都需要继承其控件MJRefreshGifHeader（自定义动画）、 MJDIYHeader（自定义布局），而且继承层数多，跨度大，由于项目庞大，灵活性差，需要统一开发口径
MJ
2. 关键检测刷新手势方法，通过KVO获取scrollVIew的参数，通过枚举定义多个状态，但是逻辑设置耦合比较厉害，修改过程略麻烦。

#### SVPullToRefresh
这是一个可能是大家比较陌生的下拉刷新控件，但是它也是有特别之处，而且github上的star数还是蛮多的，证明还是有不少人使用的。

Good

1. 添加方法简单，只需要一行代码，就能添加下拉刷新的功能

``` objc
[tableView addPullToRefreshWithActionHandler:^{
   		// 预加载数据到dataSource, 向tableview插入cell
    	// 完成时调用
    	[tableView.pullToRefreshView stopAnimating]
}];
```

Bad

1. 通过Associative（关联）为scrollView添加属性，通过KVO监测用户交互动作
2. 在默认效果中，发现 SVPullToRefreshView和控制代码写在同一个类中，代码仲有耦合比较严重，如果需要添加自定义效果，可基本需要完全重写整个SVPullToRefreshView，需要工作量较大。

## 开发阶段

要进入真正的开发阶段，令大家意外的是，我并没有选用上述的三个下拉刷新控件进行开发，而是选用的一个叫RefreshControl的控件进行开发，我下面介绍一下为什么选用这个控件。

#### RefreshControl

* 使用方式简单，只需要添加相关控件及自定义Views即可使用
* 自定义加载样式与控件完全解耦，通过注册方式可以添加任意样式视图，添加方式灵活。
* 可设置属性多，包括下拉触发位置，是否自定触发刷新
* refreshControl初始化代码如下

``` objc    
	///初始化
    _refresh=[[RefreshControl alloc]initWithScrollView:tableView delegate:self];
    ///设置显示下拉刷新
    _refresh.topEnabled=YES
    ///注册自定义的下拉刷新view
    [_refresh registerClassForTopView:[SampleRefreshView class]];
    ///设置下拉改变状态的位置
    _refresh.enableInsetTop = 65;
    ///下拉到指定位置自动刷新
    _refresh.autoRefreshTop = NO;
```
  

#### 消息提醒窗口

下面就是我们这次开发的核心阶段，根据我们从网易新闻下拉控件观察到动画效果，我归纳出几个要点。

1. 弹窗内容参数化
2. 弹窗展示动画具有bounds效果
3. refreshControl具有两段收起动画已配合弹窗出口展示
4. 在弹窗展示过程中，如果用户对相应scrollView进行手势操作，暂停弹窗展示动画进程。当用户手离开scrollView时，动画继续

###### 1.弹窗内容参数化

根据RefreshControl的特性，我们可以通过注册一个RefreshNewsView作为自定义弹窗视图，并设置相应的属性定义启用。

``` objc 
   //下拉刷新
	_refreshControl = [[RefreshControl alloc] initWithScrollView:self.fallTableView delegate:self];
   [_refreshControl registerClassForTopView:[KGRefreshTopView class]];
   	_refreshControl.sysDefaultInsetOfScroller = self.fallTableView.contentInset;
    _refreshControl.enableInsetTop = kDefaultEnableInsetTop + TabViewHeight;
    _refreshControl.refreshViewEnabled = YES;
    _refreshControl.topEnabled = YES;
    _refreshControl.bottomEnabled = YES;
```

###### 2.弹窗内容具有bounce效果

为了使消息窗口弹出动画具有bounce效果，需要两端动画实现，这里使用了参数化设置，方便嵌套调用.

``` objc
- (void)animationWithView:(UILabel *)view
                    Scale:(CGFloat)scale
                    Alpha:(CGFloat)alpha
                     Stop:(BOOL)stop
{
    view.alpha = alpha;
    view.layer.anchorPoint = CGPointMake(0.5, 0.5);
    
    [UIView animateWithDuration:0.25
                          delay:0
                        options:UIViewAnimationOptionCurveEaseIn
                     animations:^{
                         
                         view.transform = CGAffineTransformMakeScale(scale, scale);
                         view.alpha = 1;
                         
                     } completion:^(BOOL finished) {
                         
                         if (!stop && finished) {
                             [self animationWithView:view
                                               Scale:1/scale
                                               Alpha:1
                                                Stop:YES];
                         }
                     }];
    
}
```
##### 3. refreshControl具有两段收起动画已配合弹窗出口展示

通过UIViewAnimation的block方法的嵌套使用，实现多段的动画展示

``` objc
[UIView animateWithDuration:0.2 animations:^{
        
        UIEdgeInsets tmpInsets;
        if ([self isShowNewsEnableDirection:RefreshDirectionTop]) {
            tmpInsets = _sysDefaultInsetOfScroller;
            tmpInsets.top = _sysDefaultInsetOfScroller.top + self.enableRefreshHeight;
        }else{
            tmpInsets = _sysDefaultInsetOfScroller;
        }
        _scrollView.contentInset = tmpInsets;
        
        [self.topView performSelector:@selector(finishRefreshing)];
        
    } completion:^(BOOL finished) {
        
        if ([self isShowNewsEnableDirection:RefreshDirectionTop])
        {
            self.topView.hidden = YES;
            [self.refreshNewsView performSelector:@selector(showWithString:)
                                       withObject:self.newsString];
        }
        
        if ([self isShowNewsEnableDirection:RefreshDirectionTop]) {
            [UIView animateWithDuration:0.2
                                  delay:1.5
                                options:UIViewAnimationOptionAllowUserInteraction
                             animations:^{
                                 _scrollView.contentInset = _sysDefaultInsetOfScroller;
                             } completion:^(BOOL finished) {
                                 [self.refreshNewsView performSelector:@selector(dismiss)];
                                 self.topView.hidden = NO;
                                 self.newsString = nil;
                                 _refreshingDirection = RefreshingDirectionNone;
                             }];
        }else{
            _refreshingDirection = RefreshingDirectionNone;
        }
    }];
```

##### 4. 在弹窗展示过程中，如果用户对相应scrollView进行手势操作，暂停弹窗展示动画进程。当用户手离开scrollView时，动画继续

通过[苹果文档](https://developer.apple.com/library/content/qa/qa1673/)我们可以了解到，动画是可以在过程中停止可继续的，引起，我们就需要在用户开始滚动时停止我们的弹窗动画，离开时继续。

``` objc
    if (self.topEnabled && self.topView.hidden == YES && self.scrollView.isTracking && !self.isAnimationStop) {
        
        [self pasueAnimation:self.scrollView];
        
    }else if(self.topEnabled && self.topView.hidden == YES && self.scrollView.isDecelerating && self.isAnimationStop){
        
        [self resumeAnimation:self.scrollView];
    }
```
***

**到此，我们的基本问题到已经结束了，但是这个控件后续优化的控件还很大，这个只是第一版，后面我还会持续更新扩展。**

Demo：[GitHub](https://github.com/handmuch/RefreshControl_News)



