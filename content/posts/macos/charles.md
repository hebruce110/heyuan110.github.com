---
title: 网络抓包工具Charles
subtitle: 
date: 2015-08-15 11:33:11
toc: true
tags:
    - 抓包
    - charles
---


Charles是Mac下常用的网络抓包工具，常用来模拟数据和网络辅助接口调试，作为代理抓取网络请求数据，这篇文章记录了几个实用场景，希望对你有帮助。

[Charles下载传送门](http://pan.baidu.com/s/1mgszWRu)。经常会有这样的场景:

场景一: 想看看其他的App是怎样设计请求，怎样设计返回数据格式，某一个功能点请求分几个实现的。最近我在用某听书软件听鬼故事（^0^）,它们限制非VIP每天只能下100篇离线，我试着用charles拦截修改返回数据，把我自己变成超级VIP了，然后, 没然后了....

场景二: 一个请求发起直接返回各种看起来奇葩的错误，检查半天代码好像也没问题，直接就大嘴巴叫后台的兄弟服务挂了，后台一看，好好的啊...

场景三: 开发新的功能，接口也先大概定义好了，可后台兄弟忙着和妹子聊天（_^_），接口还没写好啊...,虽然可以在代码里写死demo数据，但后台接口写好了，难道又去改一遍?有木有更好的方式呢？先把请求都写好，能正常返回数据，解析好结果绑定到界面，最后接口写好了直接就对接，charles可以帮助我们这么干。

场景四: 除了WIFI我们还要测试2G,3G,4G等各种复杂网络条件下的情况，可手机上网资费不便宜啊，可以让charles限制网速模拟网络环境。

就列举这么几个场景吧，下面进入本文的正题 <!--more-->


## 一.设置代理抓包

打开charles软件,选择Proxy->Proxy Settings到如下界面:

![charles_proxysettings](/images/charles/charles_proxysettings.png)


以iPhone手机为例，打开: 设置->无线局域网，选择一个网络进入，滚动到下面看到有'HTTP代理'模块，选择手动模式，按照如下图填好配置.

![iphone_proxy](/images/charles/iphone_proxy.png)

服务器地址填写电脑的局域网ip，打开系统设置->网络，就能看到本机ip了
填写好了，随便打开一个app，在charles软件里应该会弹出一个提示框，是否同意通过本机代理上网，点是就好了。下面是我设置好代理后，打开手机上app，在charles软件里看到的。

![charles_maoyan](/images/charles/charles_maoyan.png)


## 二.模拟慢网速请求

App开发完后，我们要测试多环境，特别是在慢网速下的case，之前我有写过一篇关于[慢网速测试](http://www.heyuan110.com/2015/06/16/Mac%E6%B5%8B%E8%AF%95%E6%A8%A1%E6%8B%9F%E6%85%A2%E7%BD%91%E9%80%9F/)，现在用charles也可以达到这目的。选择Proxy->Throtting Setting，打开后如下图设置

![charles_throttling](/images/charles/charles_throttling.png)

如果要针对某一个地址限速，在Hosts里可以add要限速的url.

## 三.截获请求转到指定的地址

比如api请求的是 http://api.test.com/user?user_id=1 , 但是后台这个没写好，我们就临时转到一个本地地址 http://10.1.1.111/user_info.json ，这个就一json文件编辑什么的特别方便，想怎么改都方便了，不过这个得在本地搭一个server环境，推荐用nginx或apache.

选择Tools->Map Local打开设计界面,设置好如下图:

![charles_mapremote](/images/charles/charles_mapremote.png)

## 四.截获请求直接返回本地的文件内容

如果懒得搭server环境，就可以用这种方式了，这个可以直接把一个请求返回内容映射到本地文件，例如把 http://api.test.com/user?user_id=1 对应的请求返回内容映射到本地文件user_info.json，

选择Tools->Map Local打开设计界面,设置好如下图:

![charles_maplocal](/images/charles/charles_maplocal.png)


## 五.截获请求修改请求信息

上面的方式是直接替换了整个，哪如果只想截获并做一定修改怎么处理呢?

选择Tools->Rewrite，设置如下图:

![charles_rewrite](/images/charles/charles_rewrite.png)


## 六.设置请求的黑名单

不想某些请求发起，直接返回404，可以用黑名单

选择Tools->Black List，设置如下图:

![charles_blacklist](/images/charles/charles_blacklist.png)

## 七.DNS欺骗

dns欺骗，说简单点就是把域名解析到一个假的ip，
可以不必一定要用locahost,127.0.0.1,装个B把127.0.0.1对应到baidu.com来调试~
选择Tools->DNS Spoofing，设置如下图:
![charles_dns](/images/charles/charles_dns.png)

## 八.缓存请求返回的内容

这个我用来干过做缓存数据用，让app在没有server的时候也能跑，
选择Tools->Mirror，设置如下图:

![charles_mirror](/images/charles/charles_mirror.png)

上面这些是我在开发过程中经常会用到的，基本能很好解决和后台联调的问题，我没有把每个地方都列的很细，基本都是只提到点，相信大家知道这个点去操作都很容易就上手，但我更想说的是，这些都只是工具，最好还是能了解web原理基础，理解HTTP协议。

另外介绍一个模拟请求的工具，Chrome下的插件postman，很方便就能模拟post,get,put,delete等请求，模拟文本，上传文件请求，附上一张截图:
![post_man](/images/charles/post_man.png)

有了这些工具的辅助，相信你对接口的调试再也不会叫苦啦...

总算写完了，又晕又困，睡觉去~

===

PS:补充breakpoints调试

## 九. BreakPoints

实用的断点调试，体验就像在xcode里断点调试一样，请求时会弹出断点调试的窗口。

- 打断点

首先打开设置窗口
![post_man](/images/charles/charles_breakpoints1.png)
点击add，添加想要断点调试的url如下图
![post_man](/images/charles/charles_breakpoints2.png)
打断点完成

- 调试
 
 请求断点的url时会弹出断点调试窗口，如下图 
![post_man](/images/charles/charles_breakpoints3.png)

这里可以看到编辑request和response，如下图（这里演示的url只打了response的断点）
![post_man](/images/charles/charles_breakpoints4.png)

编辑完成点execute，搞定!


