---
layout: post
title: "Grand Central Dispatch "
date: 2013-04-22 13:46
comments: true
categories: IOS
---
<p>
Grand Central Dispatch (GCD)是Apple开发的一个多核编程的解决方法。该方法在Mac OS X 10.6雪豹中首次推出，并随后被引入到了iOS4.0中。GCD是一个替代诸如NSThread, NSOperationQueue, NSInvocationOperation等技术的很高效和强大的技术，它看起来象就其它语言的闭包(Closure)一样，但苹果把它叫做blocks
</p>

<p>
应用举例
让我们来看一个编程场景。我们要在iphone上做一个下载网页的功能，该功能非常简单，就是在iphone上放置一个按钮，点击该按钮时，显示一个转动的圆圈，表示正在进行下载，下载完成之后，将内容加载到界面上的一个文本控件中。不用GCD前
虽然功能简单，但是我们必须把下载过程放到后台线程中，否则会阻塞UI线程显示。所以，如果不用GCD, 我们需要写如下3个方法：<font color='red'>someClick</font> 方法是点击按钮后的代码，可以看到我们用NSInvocationOperation建了一个后台线程，并且放到NSOperationQueue中。后台线程执行<font color='red'>download</font>方法。
<font color='red'>download</font> 方法处理下载网页的逻辑。下载完成后用<font color='red'>performSelectorOnMainThread</font>执行<font color='red'>download_completed</font> 方法。
<font color='red'>download_completed</font> 进行clear up的工作，并把下载的内容显示到文本控件中。

这3个方法的代码如下:
</p>

{% codeblock lang:objc %}
-(IBAction)someClick:(id)sender
{ 
  self.indicator.hidden = NO; 
  [self.indicatorstart  Animating];
  queue = [[NSOperationQueue  alloc]init];
  NSInvocationOperation *op = [[[NSInvocationOperation alloc]initWithTarget:self 
                                                                   selector:@selector(download) 
                                                                     object:nil]autorelease];    
  [queueaddOperation:op];
 }

{% endcodeblock %}
<!--more-->
{% codeblock lang:objc %}
-(void)download
{
  NSURL *url = [NSURLURLWithString:@"http://www.youdao.com"];
  NSError*error;
  NSString *data =  [NSString stringWithContentsOfURL:url 
                                             encoding:NSUTF8StringEncoding
                                                error:&error];
  if(data != nil)
  {
    [self performSelectorOnMainThread:@selector(download_completed:)
                          withObject:data 
                       waitUntilDone:NO];
  }
  else
 {
    NSLog(@"error when download:%@",error);
   [queue  release];
  }
}

{% endcodeblock %}


{% codeblock lang:objc %}

-(void)download_completed:(NSString*)data
{
  NSLog(@"call back");
  [self.indicator  stopAnimating];
  self.indicator.hidden = YES;
  self.content.text = data;
  [queue  release];
}

{% endcodeblock %}

<p>使用GCD后
如果使用GCD，以上3个方法都可以放到一起，如下所示：</p>
{% codeblock lang:objc %}

- (void)download
{
   self.indicator.hidden = NO;  
   [self.indicator  startAnimating];
   dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT,0),^{ 
   NSURL*url = [NSURLURLWithString:@"http://www.youdao.com"];
   NSError *error;
   NSString *data = [NSString stringWithContentsOfURL:url
                                             encoding:NSUTF8StringEncoding
                                                error:&error];
   if(data!=nil)
   { 
      dispatch_async(dispatch_get_main_queue(),^{
      [self.indicator  stopAnimating];
      self.indicator.hidden = YES;
      self.content.text=data;});
   }
   else
   {
      NSLog(@"error when download:%@",error);
   }
   });
}

{% endcodeblock %}

<p>首先我们可以看到，代码变短了。因为少了原来3个方法的定义，也少了相互之间需要传递的变量的封装。
另外，代码变清楚了，虽然是异步的代码，但是它们被GCD合理的整合在一起，逻辑非常清晰。如果应用上MVC模式，我们也可以将View Controller层的回调函数用GCD的方式传递给Modal层，这相比以前用@selector的方式，代码的逻辑关系会更加清楚。
</p>

###GCD的定义

<p>简单GCD的定义有点象函数指针，差别是用 ^ 替代了函数指针的 * 号，如下所示：</p>
{% codeblock lang:objc %}
 // 申明变量
 (void) (^loggerBlock)(void);
 // 定义

 loggerBlock = ^{
      NSLog(@"Hello world");
 };
 // 调用
 loggerBlock();
{% endcodeblock %}

<p>但是大多数时候，我们通常使用内联的方式来定义它，即将它的程序块写在调用的函数里面，例如这样：</p>

{% codeblock lang:objc %}
dispatch_async(dispatch_get_global_queue(0, 0), ^{
      // something
 });
{% endcodeblock %}

<p>从上面大家可以看出，block有如下特点：</p>

<p>
程序块可以在代码中以内联的方式来定义。
程序块可以访问在创建它的范围内的可用的变量。
</p>

###系统提供的dispatch方法

<p>
为了方便地使用GCD，苹果提供了一些方法方便我们将block放在主线程 或 后台线程执行，或者延后执行。使用的例子如下：
</p>

{% codeblock lang:objc %}
 //  后台执行：
 dispatch_async(dispatch_get_global_queue(0, 0), ^{
      // something
 });
 // 主线程执行：
 dispatch_async(dispatch_get_main_queue(), ^{
      // something
 });
 // 一次性执行：
 static dispatch_once_t onceToken;
 dispatch_once(&onceToken, ^{
     // code to be executed once
 });
 // 延迟2秒执行：
 double delayInSeconds = 2.0;
 dispatch_time_t popTime = dispatch_time(DISPATCH_TIME_NOW, delayInSeconds * NSEC_PER_SEC);
 dispatch_after(popTime, dispatch_get_main_queue(), ^(void){
     // code to be executed on the main queue after delay
 });
{% endcodeblock %}

<p>dispatch_queue_t 也可以自己定义，如要要自定义queue，可以用dispatch_queue_create方法，示例如下：
</p>

{% codeblock lang:objc %}
dispatch_queue_t urls_queue = dispatch_queue_create("blog.devtang.com", NULL);
dispatch_async(urls_queue, ^{
     // your code
});
dispatch_release(urls_queue);
{% endcodeblock %}

<p>
另外，GCD还有一些高级用法，例如让后台2个线程并行执行，然后等2个线程都结束后，再汇总执行结果。这个可以用dispatch_group, dispatch_group_async 和 dispatch_group_notify来实现，示例如下：
</p>

{% codeblock lang:objc %}
dispatch_group_t group = dispatch_group_create();
 dispatch_group_async(group, dispatch_get_global_queue(0,0), ^{
      // 并行执行的线程一
 });
 dispatch_group_async(group, dispatch_get_global_queue(0,0), ^{
      // 并行执行的线程二
 });
 dispatch_group_notify(group, dispatch_get_global_queue(0,0), ^{
      // 汇总结果
 });
{% endcodeblock %}

###修改block之外的变量


<p>
默认情况下，在程序块中访问的外部变量是复制过去的，即写操作不对原变量生效。但是你可以加上 __block来让其写操作生效，示例代码如下：
</p>

{% codeblock lang:objc %}
 __block int a = 0;
 void  (^foo)(void) = ^{
      a = 1;
 }
 foo();
 // 这里，a的值被修改为1
{% endcodeblock %}


###后台运行

<p>
GCD的另一个用处是可以让程序在后台较长久的运行。在没有使用GCD时，当app被按home键退出后，app仅有最多5秒钟的时候做一些保存或清理资源的工作。但是在使用GCD后，app最多有10分钟的时间在后台长久运行。这个时间可以用来做清理本地缓存，发送统计数据等工作。

让程序在后台长久运行的示例代码如下：
</p>

{% codeblock lang:objc %}
// AppDelegate.h文件
@property (assign, nonatomic) UIBackgroundTaskIdentifier backgroundUpdateTask;

// AppDelegate.m文件
- (void)applicationDidEnterBackground:(UIApplication *)application
{
    [self beingBackgroundUpdateTask];
    // 在这里加上你需要长久运行的代码
    [self endBackgroundUpdateTask];
}

- (void)beingBackgroundUpdateTask
{
    self.backgroundUpdateTask = [[UIApplication sharedApplication] beginBackgroundTaskWithExpirationHandler:^{
        [self endBackgroundUpdateTask];
    }];
}

- (void)endBackgroundUpdateTask
{
    [[UIApplication sharedApplication] endBackgroundTask: self.backgroundUpdateTask];
    self.backgroundUpdateTask = UIBackgroundTaskInvalid;
}
{% endcodeblock %}

###总结
<p>
总体来说，GCD能够极大地方便开发者进行多线程编程。如果你的app不需要支持iOS4.0以下的系统，那么就应该尽量使用GCD来处理后台线程和UI线程的交互。
</p>



