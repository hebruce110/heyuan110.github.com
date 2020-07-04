---
title:      "Nginx报No route to host错误"
date: 2020-07-03T13:19:44+08:00
tags: 
  - nginx
  - linux
---


最近调整了后端站点架构为：`user------->nginx proxy server--------》internal host ------(cname)----》elb host`，跑一段时间后经常出现诡异的情况，访问时不时会挂掉，只要重启nginx proxy集群里所有机器的nginx就能恢复，
经过检查发现集群里部分机器往后转发时报如下错误:

```
 2020/06/08 16:31:20 [error] 13741#0: *116374839 connect() failed (113: No route to host) while connecting to upstream, client: 2607:xxxx:969:f1f0:c3d:70ec:178f:fd24, server: localhost, request: "POST /v1.4/source HTTP/1.1", upstream: "http://172.31.xx.xx:80/v1.4/source", host: "api.xxxx.com"
```

网上搜索找到的资料都讲防火墙原因，经过仔细排查排除了。通过仔细分析出现异常时的监控发现，在异常时间点aws elb的ip发生了变化，于是将症状和云服务的SA沟通，他说AWS ELB的IP是会变化的（之前以为是固定的），
建议nginx上的upstream需要动态去刷新，并建议使用nginx的jdomain模块来做动态解析，参考地址<https://www.nginx.com/resources/wiki/modules/domain_resolve/>，我们加上这个模块后重新部署nginx问题就没再出现。

其实原因很简单，nginx proxy指向域名时，nginx会缓存域名解析到的ip，如果域名对应的ip改变了而proxy nginx又没刷新缓存的ip，后面流量继续往旧ip转发，就会报错了，增加的动态解析模块指定dns和时间间隔去获取最新的ip并更新到缓存里，这样就保证了域名指向的ip变更proxy nginx能及时更新缓存ip。


