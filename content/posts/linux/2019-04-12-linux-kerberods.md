---
title: 挖矿病毒kerberods的入侵和处理
date: 2019-04-12 16:15:53
tags:
    - Jenkins
    - jira
    - confluence
    - 病毒入侵
    - linux
    - gitlab
---

最近经常听到挖矿病毒kerberods肆虐，大量linux主机沦陷，导致的结果显著特征CPU持续100%，正常的应用服务无法提供。不幸昨天我们有一台机器中招了，下面记录整个事件发生、处理过程。

## 事件发生

昨天下午5.30左右，几个同事反馈git代码无法提交，报502错误。

![](https://raw.githubusercontent.com/heyuan110/static-source/master/media/kerberods/15550614255551.jpg)

立即安排了一个运维童鞋排查，本来以为和上次一样gitlab并发数达到上限，改改配置重启下就行，结果从18点到20点一直顺着gitlab502错误这个方向搞了2小时，没有任何进展。由于代码无法提交，gitlab访问不了，发布系统也就不能正常做上线操作，赶紧我也投入了排查。 

## 排查处理

这台机器上有三个服务gitlab，jira，confluence，除了gitlab，另外两个看着是正常的(其实里面部分页面已经异常，只是没仔细去检查)。

1. 检查系统运行指标，先上服务器上执行`top`，如下：

![981554911843_.pic_hd](https://raw.githubusercontent.com/heyuan110/static-source/master/media/kerberods/981554911843_.pic_hd.jpg)

粗略一看cpu和内存好像正常，按1切换看每个cpu情况问题来了，居然只有一个cpu0，这台ec2机器有16CPU，接着到aws console上看到这台机器cpu，如下：

![881554911487_.pic_hd](https://raw.githubusercontent.com/heyuan110/static-source/master/media/kerberods/881554911487_.pic_hd.jpg)

5.30左右cpu突然直接100%了，其实这时脑袋闪了一下有想到是病毒，然后去看了机器的network out/in发现没有突然激增，所以没继续朝这想。随后执行`ps -ef`查看进程，发现居然没有php，java相关进程，开始想是不是aws系统故障（之前有前科）？

于是让同事赶紧给aws提了个加急support，同时联系专门对接我们的AWS SA让其帮忙跟进催促。这时我们还是没意识到是感染了病毒，如是边等待aws那边消息，边继续在google搜索gitlab502错误相关的资料。

gitlab论坛、google上能搜到资料都尝试了，基本都在说是因为端口被占用，超时时间设置等等，执行`netstat -ntlp`发现gitlab服务的8080端口压根没占用，不过抱着侥幸心理还是改了默认端口然后`gitlab-ctl reconfigure`和`gitlab-ctl restart`，最后用`gitlab-ctl status`查看发现显示所有的服务都起来了，翻遍了gitlab里相关服务所有的log，唯有有点线索的还是显示127.0.0.1:8080 refused，gitlab坛子里有人说要多尝试其次，又试了几遍，当然不会有结果啊，最后我基本确认不是gitlab问题，又去催aws sa帮我联系他们support。

中间已经开始做最坏打算，让另外一个同事开新机器把gitlab迁移过去，可是发现gitlab backup超级无敌慢，不过还是耐心的等待着。

还好大概在一个小时后美国那边的supprot打电话过来，以为有救了，nonono，接下来一度很无语，给他们描述完问题，才发现他们毛都干不了。aws规定他们售后工程师不能碰客户机器(说是基于客户安全考虑，不过应该有点扯，都托管到他们那了想干啥不行啊)，那你不能碰机器，又不能看到底层，凭超能力感知么？反正无厘头一顿指挥操作后，没啥结果。然后让做一个AMI，也是神坑速度超慢，点开始后就只能傻傻等，最后我提出来远程协助工具，居然没有，那我share屏幕，你看着然后一起分析总行吧，还好这个有，很快他发了一个screenshare的link，下载安装好bomgar，打开开始share屏幕，至此才可以说是support开始。。。。尼玛有sharescreen这种工具怎么不一上来就提出来用，唾沫星子能解决个毛线问题啊。

>工欲善其事必先利其器，这里真心要吐槽下aws support的"维修工具箱"，有顾虑和安全意识不让碰客户机器可以理解，可是能不能搞一些方便沟通提高解决问题效率的工具，比如远程协助，分享屏幕，文字、图片截图聊天等等。

待续...


## 总结




