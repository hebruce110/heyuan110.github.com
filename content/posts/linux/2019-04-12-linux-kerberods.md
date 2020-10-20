---
title: 挖矿病毒kerberods的入侵和处理
date: 2019-04-12 16:15:53
toc: true
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


以上是吐槽，得结论：16CPU的top命令只能看到1CPU在跑，有点诡异，继续排查。

2. 重启EC2

怀疑是EC2硬件故障，并且反复尝试了多次；发现上述现象依然存在，服务器始终无法显示所有的CPU资源，依旧显示一颗CPU在运行，查看系统启动日志/var/log/syslog，日志最后提示：`didn’t collect load info for all cpus, balancing is broken`


3. 排查CPU

先对故障的机器做快照，准备快照完起新机器还原过去试试。同时开始排查CPU，大概过程如下：

```

$ cat /proc/cpuinfo

查看CPU信息，系统在启动过程中，CPU的信息在启动的过程中被装载到虚拟目录/proc下的cpuinfo文件中

$ cat /var/log/syslog | grep irqbalance

查看系统日志过滤CPU中断分布信息；

关于irqbalance的更多信息：http://blog.huatai.me/2014/11/05/linux-irqbalance-irq-and-cpu-affinity/

$ cat /proc/stat             # 查看CPU的利用率

$ mpstat -P ALL 1 3 或

$ for i in `seq 3`; do cat /proc/stat >> /tmp/stat.txt; sleep 1; done

# 采集所有的CPU核心的当前运行情况，每秒更新一次，采集3次

cpu  10445020 88 14726 50034 5432 0 457 11658 0 0

cpu0 653379 0 922 2011 805 0 13 535 0 0

cpu1 651988 0 1745 2566 872 0 26 782 0 0

cpu2 653171 37 925 2511 598 0 4 735 0 0

cpu3 653520 0 807 2460 455 0 0 741 0 0

cpu4 651566 0 1390 3614 486 0 197 729 0 0

cpu5 652563 0 1318 2713 447 0 196 752 0 0

cpu6 652903 0 734 3258 340 0 1 748 0 0

cpu7 653128 0 842 2762 502 0 2 733 0 0

cpu8 652979 0 692 3446 128 0 0 733 0 0

cpu9 652710 0 729 3701 100 0 5 731 0 0

cpu10 652843 51 760 3471 119 0 3 732 0 0

cpu11 652815 0 703 3610 120 0 0 736 0 0

cpu12 652166 0 856 4120 102 0 1 734 0 0

cpu13 652992 0 777 3347 121 0 1 736 0 0

cpu14 653266 0 647 3284 40 0 1 753 0 0

cpu15 653024 0 874 3154 189 0 2 742 0 0

intr 33000247 89 9 0 0 1495 0 3 0 2 0 0 0 144 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 1742341 37454 89869 0 97 336 1786533 51288 92366 0 154 25 1729717 45438 85288 0 381 44 1709936 37017 91720 0 141 68 1896285 59966 56559 0 239 17 1890560 58187 89663 0 265 11 1703860 30700 92690 0 111 2 1706076 34320 77972 0 352 0 1735997 30031 92496 0 107 5 1707715 23861 89955 0 146 1 1700884 26785 92025 0 177 0 1697132 23744 92566 0 126 2 1788088 29854 74162 0 142 0 1781043 35672 92946 0 382 1 1707596 18470 91498 0 98 0 1698041 24493 92666 0 148 1 141 126617 481806 477683 20 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0

ctxt 9044786

btime 1554911733

processes 20719

procs_running 17

procs_blocked 0

softirq 35206004 0 26207552 1856 1006735 0 0 3292 1129827 0 6856742

cpu  10446621 88 14727 50040 5432 0 457 11658 0 0

cpu0 653480 0 922 2011 805 0 13 535 0 0

cpu1 652089 0 1745 2566 872 0 26 782 0 0

cpu2 653272 37 925 2511 598 0 4 735 0 0

cpu3 653620 0 807 2460 455 0 0 741 0 0

cpu4 651661 0 1390 3619 486 0 197 729 0 0

cpu5 652663 0 1318 2713 447 0 196 752 0 0

cpu6 653004 0 734 3258 340 0 1 748 0 0

cpu7 653228 0 842 2762 502 0 2 733 0 0

cpu8 653079 0 692 3446 128 0 0 733 0 0

cpu9 652811 0 729 3701 100 0 5 731 0 0

cpu10 652944 51 760 3471 119 0 3 732 0 0

cpu11 652915 0 703 3610 120 0 0 736 0 0

cpu12 652266 0 856 4120 102 0 1 734 0 0

cpu13 653092 0 777 3347 121 0 1 736 0 0

cpu14 653367 0 647 3284 40 0 1 753 0 0

cpu15 653124 0 874 3154 189 0 2 742 0 0

intr 33004598 89 9 0 0 1495 0 3 0 2 0 0 0 144 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 1742594 37455 89869 0 97 336 1786791 51288 92366 0 154 25 1729977 45441 85288 0 381 44 1710194 37020 91720 0 141 68 1896595 59981 56559 0 239 17 1890835 58193 89663 0 265 11 1704118 30705 92690 0 111 2 1706348 34321 77972 0 352 0 1736251 30033 92496 0 107 5 1707971 23863 89955 0 146 1 1701145 26786 92025 0 177 0 1697391 23748 92566 0 126 2 1788355 29855 74162 0 142 0 1781306 35675 92946 0 382 1 1707855 18471 91498 0 98 0 1698298 24496 92666 0 148 1 141 126617 481820 477693 20 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0

......

查看CPU时钟频率

$ getconf CLK_TCK

100

Hertz(tick per second)就是CLK_TCK，可以根据getconf CLK_TCK获取

$ echo l > /proc/sysrq-trigger

Send a SIGKILL to all processes, except for init

# 将所有的CPU正在执行进程打在了dmesg中

$ dmesg

文件：dmesg1.txt

关闭系统透明大页面

$ cat /sys/kernel/mm/transparent_hugepage/enabled

$ cat /sys/kernel/mm/transparent_hugepage/defrag

always [madvise] never

$ echo never > /sys/kernel/mm/transparent_hugepage/enabled

$ echo never > /sys/kernel/mm/transparent_hugepage/defrag

$ vi /etc/default/grub

$ GRUB_CMDLINE_LINUX="transparent_hugepage=never"

$ source /etc/default/grub

$ update-grub

$ cat /sys/kernel/mm/transparent_hugepage/enabled

$ cat /sys/kernel/mm/transparent_hugepage/enabled

always madvise [never]

$ echo l > /proc/sysrq-trigger

文件：dmesg2.txt

$ ps -ef | grep khugepaged

$ cd /proc

$ for f in `find . -name '[1-9][0-9]*' -maxdepth 1`; do echo $f; cat $f/comm; done

# 查看当前系统下所有正在执行的应用程序

find: warning: you have specified the -maxdepth option after a non-option argument -name, but options are not positional (-maxdepth affects tests specified before it as well as those specified after it).  Please specify options before other arguments.

./10

watchdog/0

./11

watchdog/1

./12

migration/1

./13

ksoftirqd/1

./15

kworker/1:0H

./16

watchdog/2

./17

migration/2

./18

ksoftirqd/2

./20

kworker/2:0H

./21

watchdog/3

./22

migration/3

./23

ksoftirqd/3

./25

kworker/3:0H

./26

watchdog/4

./27

migration/4

./28

ksoftirqd/4

./30

kworker/4:0H

./31

watchdog/5

./32

migration/5

./33

ksoftirqd/5

./35

kworker/5:0H

./36

watchdog/6

./37

migration/6

./38

ksoftirqd/6

./40

kworker/6:0H

./41

watchdog/7

./42

migration/7

./43

ksoftirqd/7

./45

kworker/7:0H

......

$ echo l > /proc/sysrq-trigger

文件：dmesg3.txt

[  177.581904] Sending NMI to all CPUs:

[  177.583305] NMI backtrace for cpu 0

[  177.583307] CPU: 0 PID: 2495 Comm: khugepageds Not tainted 4.4.0-145-generic #171-Ubuntu

[  177.583309] Hardware name: Xen HVM domU, BIOS 4.2.amazon 08/24/2006

 ....

[  177.583334] NMI backtrace for cpu 1

[  177.583336] CPU: 1 PID: 2492 Comm: khugepageds Not tainted 4.4.0-145-generic #171-Ubuntu

[  177.583338] Hardware name: Xen HVM domU, BIOS 4.2.amazon 08/24/2006

 .....

[  177.583363] NMI backtrace for cpu 2

[  177.583365] CPU: 2 PID: 2487 Comm: khugepageds Not tainted 4.4.0-145-generic #171-Ubuntu

[  177.583367] Hardware name: Xen HVM domU, BIOS 4.2.amazon 08/24/2006

 ......

[  177.583392] NMI backtrace for cpu 3

[  177.583394] CPU: 3 PID: 2493 Comm: khugepageds Not tainted 4.4.0-145-generic #171-Ubuntu

[  177.583395] Hardware name: Xen HVM domU, BIOS 4.2.amazon 08/24/2006

......

[  177.583421] NMI backtrace for cpu 4

[  177.583422] CPU: 4 PID: 2411 Comm: bash Not tainted 4.4.0-145-generic #171-Ubuntu

[  177.583424] Hardware name: Xen HVM domU, BIOS 4.2.amazon 08/24/2006

......

[  177.583482] NMI backtrace for cpu 5

[  177.583484] CPU: 5 PID: 2497 Comm: khugepageds Not tainted 4.4.0-145-generic #171-Ubuntu

[  177.583486] Hardware name: Xen HVM domU, BIOS 4.2.amazon 08/24/2006

......

[  177.583511] NMI backtrace for cpu 6

[  177.583513] CPU: 6 PID: 2496 Comm: khugepageds Not tainted 4.4.0-145-generic #171-Ubuntu

[  177.583514] Hardware name: Xen HVM domU, BIOS 4.2.amazon 08/24/2006

......

[  177.583540] NMI backtrace for cpu 7

[  177.583541] CPU: 7 PID: 2494 Comm: khugepageds Not tainted 4.4.0-145-generic #171-Ubuntu

[  177.583543] Hardware name: Xen HVM domU, BIOS 4.2.amazon 08/24/2006

......

[  177.583568] NMI backtrace for cpu 8

[  177.583570] CPU: 8 PID: 2501 Comm: khugepageds Not tainted 4.4.0-145-generic #171-Ubuntu

[  177.583572] Hardware name: Xen HVM domU, BIOS 4.2.amazon 08/24/2006

......

[  177.583597] NMI backtrace for cpu 9

[  177.583599] CPU: 9 PID: 2488 Comm: khugepageds Not tainted 4.4.0-145-generic #171-Ubuntu

[  177.583601] Hardware name: Xen HVM domU, BIOS 4.2.amazon 08/24/2006

......

[  177.583626] NMI backtrace for cpu 10

[  177.583628] CPU: 10 PID: 2490 Comm: khugepageds Not tainted 4.4.0-145-generic #171-Ubuntu

[  177.583629] Hardware name: Xen HVM domU, BIOS 4.2.amazon 08/24/2006

......

[  177.583655] NMI backtrace for cpu 11

[  177.583656] CPU: 11 PID: 2491 Comm: khugepageds Not tainted 4.4.0-145-generic #171-Ubuntu

[  177.583658] Hardware name: Xen HVM domU, BIOS 4.2.amazon 08/24/2006

......

[  177.583684] NMI backtrace for cpu 12

[  177.583685] CPU: 12 PID: 2486 Comm: khugepageds Not tainted 4.4.0-145-generic #171-Ubuntu

[  177.583687] Hardware name: Xen HVM domU, BIOS 4.2.amazon 08/24/2006

......

[  177.583712] NMI backtrace for cpu 13

[  177.583714] CPU: 13 PID: 2500 Comm: khugepageds Not tainted 4.4.0-145-generic #171-Ubuntu

[  177.583716] Hardware name: Xen HVM domU, BIOS 4.2.amazon 08/24/2006

......

[  177.583741] NMI backtrace for cpu 14

[  177.583743] CPU: 14 PID: 2499 Comm: khugepageds Not tainted 4.4.0-145-generic #171-Ubuntu

[  177.583744] Hardware name: Xen HVM domU, BIOS 4.2.amazon 08/24/2006

......

[  177.583770] NMI backtrace for cpu 15

[  177.583772] CPU: 15 PID: 2489 Comm: khugepageds Not tainted 4.4.0-145-generic #171-Ubuntu

[  177.583773] Hardware name: Xen HVM domU, BIOS 4.2.amazon 08/24/2006

......

从上述日志可以看出 khugepageds 分别在0、1、2、3、5、6、7、8、9、10、11、12、13、14、15上执行，唯独没有在CPU 的第4个core上存在，与top命令显示只有一个CPU在运行吻合

从上面的日志中，根据cpu13中的PID找出对应的应用程序

$ ls /proc | grep 2500

khugepageds

[ 1035.043542] NMI backtrace for cpu 6

[ 1035.043544] CPU: 6 PID: 2493 Comm: khugepageds Not tainted 4.4.0-145-generic #171-Ubuntu

[ 1035.043545] Hardware name: Xen HVM domU, BIOS 4.2.amazon 08/24/2006

。。。。。。

```

根据"khugepageds"关键字google搜索出相关资料,发现居然是和一个挖矿病毒有关，此病毒相关资料：

<https://stackoverflow.com/questions/55318938/jenkins-high-cpu-usage-khugepageds>

<https://laucyun.com/17e194c26e4554cab975aae760bad553.html>

最后查出是confluence漏洞导致

<https://community.atlassian.com/t5/Confluence-discussions/khugepageds-eating-all-of-the-CPU/td-p/1055337>

<https://confluence.atlassian.com/doc/confluence-security-advisory-2019-03-20-966660264.html?_ga=2.65341993.660720501.1556073417-575823134.1554929090>

从上面的URL得知因为confluence漏洞导致感染挖矿病毒，知道是什么病毒就好解决了，继续google找到清理这个病毒的资料：

<https://git.laucyun.com/security/lsd_malware_clean_tool/blob/master/README.md>

<https://git.laucyun.com/security/lsd_malware_clean_tool/blob/master/clear_kerberods.sh>

至此，按上面文档清理病毒，打上confluence漏洞补丁，搞定问题。

## 总结

开源的一些软件漏洞都比较多（wordpress。。。），注意关注官方重大提醒。
最后强调，备份，备份，备份！







