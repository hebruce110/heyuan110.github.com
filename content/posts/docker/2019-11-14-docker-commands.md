---
title:      "Docker常用命令"
date:       2019-11-14 20:37:13
author:     "bruce"
toc: true
tags:
    - docker
    - linux
---

Docker常用命令记录

<!-- more -->

## 1. 启动、重启或停止docker服务

启动： `service docker start`

停止：`service docker stop`

重启：`service docker restart`

## 2.镜像(images)

### 获取镜像

docker pull [OPTIONS] NAME[:TAG|@DIGEST]

例如：`docker pull ubuntu:16.04`

### 列出镜像

- 查看所有镜像: `docker images`或`docker image ls`, 显示摘要`docker images --digests`
- 查看特定镜像:`docker image ls xxxx`或`docker image xxxx`
- 查看镜像、容器、数据卷所占用的空间:`docker system df`
- 列出虚悬镜像（仓库名和标签名为<none>）:`docker image ls -f dangling=true`,清楚此类镜像`docker image prune`

### 删除镜像

命令`docker image rm [OPTIONS] IMAGE [IMAGE...]` 或 `docker rmi -f xxxx`
例如：

```
root@ubuntu:~# docker image ls redis
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
redis               latest              ce25c7293564        5 days ago          95MB
```
- 根据完整ID删除：`docker image rm 578c3e61a98c`
- 根据短ID删除：`docker image rm 578c3`
- 根据镜像名删除：`docker image rm redis`
- docker image ls配合删除：删除所有镜像`docker image rm $(docker image ls -q)`,删除所有仓库名为redis的镜像`docker image rm $(docker image ls -q redis)`
- 删除所有未打 dangling 标签的镜像`docker rmi $(docker images -q -f dangling=true)`

docker ps -a | grep "Exited" | awk '{print $1 }'|xargs docker stop
docker ps -a | grep "Exited" | awk '{print $1 }'|xargs docker rm
docker images|grep none|awk '{print $3 }'|xargs docker rmi


## 3. 容器(container)

### 创建新容器

- 使用docker镜像nginx:latest以后台模式启动一个容器,并将容器命名为mynginx。

```
root@ubuntu:~# docker run --name mynginx -d nginx:latest
参数：
--name="nginx-lb": 为容器指定一个名称；
-d: 后台运行容器，并返回容器ID；
```

- 使用镜像nginx:latest以后台模式启动一个容器,并将容器的80端口映射到主机随机端口。

```
root@ubuntu:~# docker run -P -d nginx:latest
参数：
-p: 端口映射，格式为：主机(宿主)端口:容器端口;
```

- 使用镜像 nginx:latest，以后台模式启动一个容器,将容器的 80 端口映射到主机的 80 端口,主机的目录 /data 映射到容器的 /data。

```
root@ubuntu:~# docker run -p 80:80 -v /data:/data -d nginx:latest
参数：
-v: 映射数据卷,格式为：主机目录:容器目录，注意是绝对路径;
```
- 绑定容器的 8080 端口，并将其映射到本地主机 127.0.0.1 的 80 端口上。

```
root@ubuntu:~# docker run -p 127.0.0.1:80:8080/tcp ubuntu bash
```
- 使用镜像nginx:latest以交互模式启动一个容器,在容器内执行/bin/bash命令。
```
root@ubuntu:~# docker run -it nginx:latest /bin/bash
```

参考<http://www.runoob.com/docker/docker-run-command.html>

### 其他命令：

```
//查看所有容器
docker ps -a

//查看正在运行的容器
docker ps

//启动一个容器
docker start xxxx

//停止一个容器
docker stop xxxx

//杀死所有正在运行的容器
docker kill $(docker ps -a -q)

//重启一个容器
docker restart xxxx

//删除一个容器，前提是容器是stop
docker rm xxxx

//删除所有已停止容器，慎用！
docker rm $(docker ps -a -q)

//容器与主机之间的数据拷贝
将主机/www/xxxx目录拷贝到容器test-container的/www目录下：
docker cp /www/xxxx test-container:/www/
或
将主机/www/xxxx目录拷贝到容器test-container中，目录重命名为www：
docker cp /www/xxxx test-container:/www

//查看容器ip
方法1：进入容器内部，然后cat /etc/hosts
方法2：docker inspect 容器id
```

## 常用docker

### 1. Grafana

```
//创建相关目录，和配置文件
//下载默认配置模板grafana.ini,修改defaults.ini为grafana.ini
`wget https://raw.githubusercontent.com/grafana/grafana/master/conf/defaults.ini`

//创建grafana容器
 docker run -d --name=grafana \
 -p 3000:3000 \
 -v /usr/local/programs/grafana/data:/var/lib/grafana \
 -v /usr/local/programs/grafana/log:/var/log/grafana \
 -v /usr/local/programs/grafana/conf/grafana.ini:/etc/grafana/grafana.ini \
 -e "GF_SERVER_ROOT_URL=http://s1.s:3000" \
 -e "GF_SECURITY_ADMIN_PASSWORD=123456" \
 docker.patpat.vip:9503/grafana:1.20.0:5.4.2
```

### 2. Portainer

```
docker run -d -p 9000:9000 \
    --restart=always \
    -v /var/run/docker.sock:/var/run/docker.sock \
    --name portainer \
   docker.patpat.vip:9503/portainer:1.20.0
```

### 3. Prometheus

```
//下载默认配置
wget https://raw.githubusercontent.com/prometheus/prometheus/master/documentation/examples/prometheus.yml

//创建prometheus镜像
docker run -d --name=prometheus \
    -p 9092:9090 \
    -v /usr/local/programs/prometheus/prometheus.yml:/etc/prometheus/prometheus.yml \
    -v /usr/local/programs/prometheus/data:/prometheus-data \
    prom/prometheus

//创建带alert规则的prometheus镜像  
docker run -d --name=prometheus \
    -p 9092:9090 \
    -v /usr/local/programs/prometheus/prometheus.yml:/etc/prometheus/prometheus.yml \
    -v /usr/local/programs/prometheus/alert.rules:/etc/prometheus/alert.rules \
    -v /usr/local/programs/prometheus/data:/prometheus-data \
    prom/prometheus
    
//创建alertmanager
docker run -d --name=alertmanager \
    -p 9093:9093 \
    -v /usr/local/programs/prometheus/alertmanager/config.yml:/etc/alertmanager/config.yml \
    prom/alertmanager
    
```

### 4. Mysql

启动mysql5.6镜像
```
docker run -d \
-p 3306:3306 \
--name mysql \
-v /usr/local/programs/mysql/conf:/etc/mysql/conf.d \
-v /usr/local/programs/mysql/logs:/logs \
-v /usr/local/programs/mysql/data:/var/lib/mysql \
-e MYSQL_ROOT_PASSWORD=root \
docker.patpat.vip:9503/mysql:5.6.46
```

启动mysql5.7镜像
```
docker pull docker.patpat.vip:9503/mysql:5.7.28

docker run -d \
-p 3306:3306 \
--name mysql \
-v $PWD/mysql:/var/lib/mysql \
-e MYSQL_ROOT_PASSWORD=root \
docker.patpat.vip:9503/mysql:5.7.28
```
说明:
-p 3306:3306：将容器的3306端口映射到主机的3306端口；
-v $PWD/mysql:/var/lib/mysql：将主机当前目录下的/mysql挂载到容器的/var/lib/mysql；
-e MYSQL_ROOT_PASSWORD=password：初始化root用户的密码；
--name 给容器命名，mysql5719；
-d 表示容器在后台运行

### 5. PHP

```
docker run -d -p 9091:9000 --name webconsole-php \
-v /var/www/webconsole:/var/www/html/ \
-v /usr/local/programs/php7/php7-fpm/conf:/usr/local/etc/php \
-v /usr/local/programs/php7/php7-fpm/logs:/phplogs \
--privileged=true \
docker.patpat.vip:9503/php:7.2-fpm
```

### 6. Nginx

docker run -d -p 8001:80 --name webconsole-nginx \
-v /var/www/webconsole:/usr/share/nginx/html \
-v /usr/local/programs/nginx/conf.d:/etc/nginx/conf.d \
-v /usr/local/programs/nginx/log:/var/log/nginx \
--privileged=true -d docker.patpat.vip:9503/nginx:1.15


### 7. Mongo

```
docker run -d \
--name mongodb \
-p 32767:27017 \
-v /usr/local/programs/mongodb:/data/db \
docker.patpat.vip:9501/mongo:4.2.1
```

windows下会有写入权限问题，所以需要先创建volume，再映射

`docker volume create --name=mongodata`

`docker run -d --name mongodb -p 32767:27017 -v mongodata:/data/db docker.patpat.vip:9503/mongo:4.2.1`

`docker volume`创建卷之后怎么查看？

https://stackoverflow.com/questions/44358328/how-i-can-access-docker-data-volumes-on-windows-machine

查看所有卷 `docker volume ls`

查看具体某一个卷 `docker volume inspect xxxx`

创建一个临时的环境，将docker跟目录挂载进去，登录进去后ls /var-root就可以查看所有路径文件了

`docker run --rm -it -v /:/vm-root alpine:edg sh`

### 8. Redis

```
docker run --name redis -d -p 6379:6379 -v redis-data:/data docker.patpat.vip:9503/redis:5.0.3
```


