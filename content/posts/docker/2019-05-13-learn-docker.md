---

title:      "Docker学习笔记"
date:       2019-05-13 20:33:33
author:     "bruce"
toc: true
tags:
    - docker
    - linux
---


Docker 是一个开源的应用容器引擎，基于 Go 语言 并遵从Apache2.0协议开源。Docker 可以让开发者打包他们的应用以及依赖包到一个轻量级、可移植的容器中，然后发布到任何流行的 Linux 机器上，也可以实现虚拟化。

## 1.Docker 包括三个基本概念

- 镜像(Image) 

- 容器(Container) 

- 仓库(Repository)


## 2.镜像(Image) 

Docker 镜像是一个特殊的文件系统，除了提供容器运行时所需的程序、库、资 源、配置等文件外，还包含了一些为运行时准备的一些配置参数(如匿名卷、环境 变量、用户等)。镜像不包含任何动态数据，其内容在构建之后也不会被改变。

镜像并非是像一个 ISO 那样的打包文件，镜像只是一个虚拟的概念，其 实际体现并非由一个文件组成，而是由一组文件系统组成，或者说，由多层文件系 统联合组成。

镜像构建时，会一层层构建，前一层是后一层的基础。每一层构建完就不会再发生改变，后一层上的任何改变只发生在自己这一层。在构建镜像的时候，需要额外小心，每一层尽量只包含该层需要添加的东西，任何额外的东西应该在该层构建结束前清理掉。

## 3.Docker 容器

镜像(Image)和容器(Container)的关系，就像是面向对象程序设计中的类和实例一样，镜像是静态的定义，容器是镜像运行时的实体。容器可以被 创建、启动、停止、删除、暂停等。

容器也是分层存储。每一个容器运行时，是以镜像为基础层，在其上创建一个当前容器的存储层，我们可以称这个为容器运行时读写而准备的存储层为容器存储层。

容器存储层的生存周期和容器一样，容器消亡时，容器存储层也随之消亡。因此，任何保存于容器存储层的信息都会随容器删除而丢失。容器不应该向其存储层内写入任何数据，容器存储 层要保持**无状态化**。所有的文件写入操作，都应该使用 数据卷(Volume)、或者 绑定宿主目录，在这些位置的读写会跳过容器存储层，直接对宿主(或网络存储)发 生读写，其性能和稳定性更高。

## 4.Docker Registry 

镜像构建完成后，可以很容易的在当前宿主上运行，但是，如果需要在其它服务器 上使用这个镜像，我们就需要一个集中的存储、分发镜像的服务，Docker Registry 就是这样的服务。
一个 Docker Registry 中可以包含多个仓库(Repository);每个仓库可以包含多 个标签(Tag);每个标签对应一个镜像。

## 5. Docker commit

docker commit 的语法格式为:

    docker commit [选项] <容器ID或容器名> [<仓库名>[:<标签>]]

例如：

```
$ docker commit \
--author "Tao Wang <twang2218@gmail.com>" \ --message "修改了默认网页" \
webserver \
nginx:v2
sha256:07e33465974800ce65751acc279adc6ed2dc5ed4e0838f8b86f0c87aa
1795214
```

docker commit会让制作的image越来越臃肿，不方便追溯，所以一般不用来制作镜像。commit命令除了学习之外，还有一些特殊的应用场合，比如被入侵后保 存现场等。但是，不要使用 docker commit 定制镜像，定制行为应该使用Dockerfile 来完成

## 6. Dockerfile

Dockerfile 是一个文本文件，其内包含了一条条的指令(Instruction)，每一条指令 构建一层(修改、安装、构建、操作)，因此每一条指令的内容，就是描述该层应当如何构建。

- FROM 指定基础镜像

FROM就是指定基础镜像，因此一个 Dockerfile 中 FROM 是必备的指令，并 且必须是第一条指令。

FROM scratch 表示一个空白的镜像

- RUN 执行命令

RUN 指令是用来执行命令行命令的。由于命令行的强大能力， RUN 指令在定制
镜像时是最常用的指令之一。其格式有两种:

shell 格式: `RUN <命令>`,就像直接在命令行中输入的命令一样, 例如

```
RUN echo '<h1>Hello, Docker!</h1>' > /usr/share/nginx/html/index
.html
```

exec 格式: `RUN ["可执行文件", "参数1", "参数2"] `,这更像是函数调用中 的格式。

**每个RUN都会产生一个新的层。撰写 Dockerfile 的时候，要经常提醒自 己，这并不是在写 Shell 脚本，而是在定义每一层该如何构建。**

为了格式化还进行了换行。Dockerfile 支持 Shell 类的行尾添加 \ 的 命令换行方式，以及行首 # 进行注释的格式。良好的格式，比如换行、缩进、注 释等，会让维护、排障更为容易

- 构建镜像

docker build 命令进行镜像构建。其格式为:

    docker build [选项] <上下文路径/URL/->

例如进到Dockerfile文件所在目录执行: `docker build -t nginx:v3 .`

- 镜像构建上下文(Context)

docker build 的工作原理：Docker 在运行时分为 Docker 引擎 (也就是服务端守护进程)和客户端工具。Docker 的引擎提供了一组 REST API， 被称为 Docker Remote API，而如 docker 命令这样的客户端工具，则是通过这组 API 与 Docker 引擎交互，从而完成各种功能。因此，虽然表面上我们好像是在 本机执行各种 docker 功能，但实际上，一切都是使用的远程调用形式在服务端 (Docker 引擎)完成。

当构建的时候，用户会指定构建镜像上下文的路径， docker build 命令得知这个路径后，会将路径下的所有内容打包，然后上 传给 Docker 引擎。这样 Docker 引擎收到这个上下文包后，展开就会获得构建镜 像所需的一切文件。

如果在 Dockerfile 中这么写:

    COPY ./package.json /app/

这并不是要复制执行`docker build`命令所在的目录下的 package.json ，不是复制Dockerfile所在目录下的package.json ，而是复制 上下文(context)目录下的package.json。

![](https://raw.githubusercontent.com/heyuan110/static-source/master/media/15575697047773/15576404978427.jpg)

docker build还支持从git库，tar压缩包，标准输入等方式构建镜像。

- COPY 复制文件

格式：`COPY <源路径>... <目标路径>` 和 ` COPY ["<源路径1>",... "<目标路径>"]`

两种格式，一种类似于命令行，一种类似于函数调用。

COPY指令<>将从**构建上下文目录**中 <源路径> 的文件/目录复制到新的一层的镜像内的<目标路径>位置。比如:

     COPY package.json /usr/src/app/

  
<源路径>可以是单个也可以是多个，也可以是通配符，例如

```
COPY hom* /mydir/
COPY hom?.txt /mydir/
```

<目标路径> 可以是容器内的绝对路径，也可以是相对于工作目录的相对路径(工 作目录可以用 WORKDIR 指令来指定)。目标路径不需要事先创建，如果目录不存 在会在复制文件前先行创建缺失目录。

COPY 指令，源文件的各种元数据都会保留。比如 读、写、执行权限、文件变更时间等。

- ADD 更高级的复制文件

ADD 指令和 COPY 的格式和性质基本一致。但是在 COPY 基础上增加了一些
功能。

1. <源路径> 可以是一个 URL, Docker引擎会自动下载放到目标路径，权限600，如果要解压或修改权限需再用RUN增加一层处理
2.  <源路径> 是tar/gzip/bzip2等压缩文件，ADD指令会自动解压文件到目标路径。如果真的希望是把压缩文件拷贝到目标路径，而不被自动解压，这时不能使用ADD

ADD包含更复杂的功能。ADD指令会导致镜像构建缓存时效导致构建缓慢。
尽可能使用COPY，语义明确，就是复制。除非需要自动解压缩的场景才用ADD.

- CMD 容器启动命令

Docker不是虚拟机，容器就是进程。既然是进程，那么在启动容器的时候，需要指定所运行的程序及参数。 CMD 指令就是用于指定默认的容器主进程的启动命令的。

Docker 不是虚拟机，容器中的应用都应该以前台执行，而不是像虚拟机、物理机 里面那样，用 upstart/systemd 去启动后台服务，容器内没有后台服务的概念。

CMD 指令的格式
![](https://raw.githubusercontent.com/heyuan110/static-source/master/media/15575697047773/15577263039629.jpg)

如果使用 shell 格式的话，实际的命令会被包装为 sh -c 的参数的形式进行 执行。

例如使用`CMD service nginx start`启动nginx服务会发现执行完成，容器立即就退出。是因为使用service nginx start命令，则是希望以后台守护进程形式启动nginx服务，而刚才说的`CMD service nginx start`会被理解为
会被理解为 CMD ["sh", "-c", "service nginx start"], 因此主进程实际上是sh。那么当命令结束后，sh也就结束了， sh作为主进程退出了，自然就会令容器退出。所以正确的做法应该是直接执行 nginx 可执行文件，并且要求以前台形式运行:`CMD ["nginx", "-g", "daemon off;"]`

- ENTRYPOINT 入口点

ENTRYPOINT和CMD一样都是在指定容器启动程序及参数，不过比CMD繁琐一点，需要通过docker run的参数`--entrypoint`来指定

当指定ENTRYPOINT后，CMD 的含义就发生了改变，不再是直接的运行其命令，而是将CMD做为内容传给ENTRYPOINT指令，实际执行时将变为：
`<ENTRYPOINT> "<CMD>"`

既生亮何生瑜，为什么有了CMD又有ENTRYPOINT呢？

1. ENTRYPOINT可以让镜像变成像命令一样使用
![](https://raw.githubusercontent.com/heyuan110/static-source/master/media/15575697047773/15577305262576.jpg)

假如我们使用 docker build -t myip . 来构建镜像的话，如果我们需要查询当 前公网 IP，只需要执行:`docker run myip`
那如果要在后面加-I参数呢，用`docker run myip -I`会报错，因为跟在镜像后面的是command，运行时会替换默认command，显然单独一个-I无法运行。

那如果把CMD换成ENTRYPOINT会怎样呢

![](https://raw.githubusercontent.com/heyuan110/static-source/master/media/15575697047773/15577306857517.jpg)

再执行`docker run myip -I`不会报错，因为当ENTRYPOINT存在后，CMD 的内容将会作为参数传给ENTRYPOINT ，而这里-I就是新的CMD.

2. ENTRYPOINT可以让应用运行前的准备工作

启动容器就是启动主进程，但有些时候，启动主进程前，需要一些准备工作。

- ENV 设置环境变量

![](https://raw.githubusercontent.com/heyuan110/static-source/master/media/15575697047773/15577308032517.jpg)

设置环境变量而已，无论是后面的其它指令，如 RUN ，还 是运行时的应用，都可以直接使用这里定义的环境变量。

例如node镜像的dockerfile:

![](https://raw.githubusercontent.com/heyuan110/static-source/master/media/15575697047773/15577391077696.jpg)

在这里先定义了环境变量 NODE_VERSION ，其后的 RUN 这层里，多次使用$NODE_VERSION 来进行操作定制。可以看到，将来升级镜像构建版本的时候，只
需要更新$NODE_VERSION即可，Dockerfile 构建维护变得更轻松了。

![](https://raw.githubusercontent.com/heyuan110/static-source/master/media/15575697047773/15577398706795.jpg)

- ARG 构建参数

格式： ARG <参数名>[=<默认值>]

构建参数和 ENV 的效果一样，都是设置环境变量。所不同的是， ARG 所设置的 构建环境的环境变量，在将来容器运行时是会存在这些环境变量的. 不要用来存密码，docker history可以查看到这些变量。

The ARG instruction defines a variable that users can pass at build-time to the builder with the docker build command using the --build-arg <varname>=<value> flag.
ARG指令定义了用户可以在编译时或者运行时传递的变量，如使用如下命令：--build-arg <varname>=<value>

The ENV instruction sets the environment variable <key> to the value <value>. The environment variables set using ENV will persist when a container is run from the resulting image.
ENV指令是在dockerfile里面设置环境变量，不能在编译时或运行时传递。

- VOLUME 定义匿名卷

格式为: ` VOLUME ["<路径1>", "<路径2>"...]` 或 ` VOLUME <路径>`

容器运行时应该尽量保持容器存储层不发生写操作，对于数据库类 需要保存动态数据的应用，其数据库文件应该保存于卷(volume)中。

`VOLUME /data` 这里的 /data 目录就会在运行时自动挂载为匿名卷，任何向 /data 中写入的 信息都不会记录进容器存储层，从而保证了容器存储层的无状态化。

运行时 可以覆盖这个挂载设置。比如: ` docker run -d -v mydata:/data xxxx`, 在这行命令中，就使用了 mydata 这个命名卷挂载到了 /data 这个位置，替代 了 Dockerfile 中定义的匿名卷的挂载配置。

- EXPOSE 声明端口

格式为 EXPOSE <端口1> [<端口2>...]

EXPOSE 指令是声明运行时容器提供服务端口，这只是一个声明，在运行时并不 会因为这个声明应用就会开启这个端口的服务

要将 EXPOSE 和在运行时使用 -p <宿主端口>:<容器端口> 区分开来。 -p是映射宿主端口和容器端口，换句话说，就是将容器的对应端口服务公开给外界访问，而 EXPOSE 仅仅是声明容器打算使用什么端口而已，并不会自动在宿主进行 端口映射。

- WORKDIR 指定工作目录

格式为 WORKDIR <工作目录路径> 。

使用 WORKDIR 指令可以来指定工作目录(或者称为当前目录)，以后各层的当前 目录就被改为指定的目录，如该目录不存在， WORKDIR 会帮你建立目录。

初学者常犯的错误是把 Dockerfile 等同于 Shell 脚本来书写，例如下面的错误：

```
RUN cd /app
RUN echo "hello" > world.txt
```

如果将这个 Dockerfile 进行构建镜像运行后，会发现找不到 /app/world.txt 文 件，或者其内容不是 hello。原因其实很简单，在 Shell 中，连续两行是同一个 进程执行环境，因此前一个命令修改的内存状态，会直接影响后一个命令;而在 Dockerfile 中，这两行 RUN 命令的执行环境根本不同，是两个完全不同的容器（两个RUN在两个层）。

每一个RUN 都是启动一个容器、执行命令、然后提交存储层文件变更。
第一层 的执行仅仅是当前进程的工作目录变更，一个内存上的变 化而已，其结果不会造成任何文件变更。而到第二层的时候，启动的是一个全新的 容器，跟第一层的容器更完全没关系，自然不可能继承前一层构建过程中的内存变 化。

因此如果需要改变以后各层的工作目录的位置，那么应该使用 WORKDIR 指令。

- USER 指定当前用户

![](https://raw.githubusercontent.com/heyuan110/static-source/master/media/15575697047773/15577429025013.jpg)

- HEALTHCHECK 健康检查

![](https://raw.githubusercontent.com/heyuan110/static-source/master/media/15575697047773/15577489951500.jpg)

HEALTHCHECK指令告诉 Docker 应该如何进行判断容器的状态是否正常

![](https://raw.githubusercontent.com/heyuan110/static-source/master/media/15575697047773/15577490221766.jpg)

![](https://raw.githubusercontent.com/heyuan110/static-source/master/media/15575697047773/15577490365000.jpg)

命令的返回值决定了该次健康检查的成功与否，0成功，1失败，2保留。

- ONBUILD 为他人做嫁衣裳

格式: ONBUILD <其它指令>

ONBUILD 是一个特殊的指令，它后面跟的是其它指令，比如 RUN , COPY 等， 而这些指令，在当前镜像构建时并不会被执行。只有当以当前镜像为基础镜像，去 构建下一级镜像的时候才会被执行。

## 7. 迁移镜像

Docker 还提供了 docker load 和 docker save 命令，用以将镜像保存为一 个 tar 文件，然后传输到另一个位置上，再加载进来。

这个是在没有docker registry时的做法，现在已经不推荐（可用docker hub或私网registry），但在有些场景还是可以用。

- docker save导出镜像

例如：` $ docker save alpine | gzip > alpine-latest.tar.gz`

把alpine镜像导出成 alpine-latest.tar.gz

- docker load导入镜像

用命令`docker load -i alpine-latest.tar.gz`导入镜像

导出和导入结合ssh可以很好的做一些镜像迁移。

## 8. 删除镜像

`docker rmi [选项] <镜像1> [<镜像2> ...]`

注意 docker rm 命令是删除容器，不要混淆。

镜像删除行为分为两类，一 类是 Untagged ，另一类是 Deleted
镜像的唯一标识是其 ID 和摘要，而一个镜像可以有多个标签。

因此当我们使用上面命令删除镜像的时候，实际上是在要求删除某个标签的镜像。
所以首先需要做的是将满足我们要求的所有镜像标签都取消，这就是我们看到的
Untagged 的信息。因为一个镜像可以对应多个标签，因此当我们删除了所指定 的标签后，可能还有别的标签指向了这个镜像，如果是这种情况，那么 Delete 行为就不会发生。所以并非所有的 docker rmi 都会产生删除镜像的行为，有可 能仅仅是取消了某个标签而已。

当该镜像所有的标签都被取消了，该镜像很可能会失去了存在的意义，因此会触发 删除行为。镜像是多层存储结构，因此在删除的时候也是从上层向基础层方向依次 进行判断删除。镜像的多层结构让镜像复用变动非常容易，因此很有可能某个其它 镜像正依赖于当前镜像的某一层。这种情况，依旧不会触发删除该层的行为。直到 没有任何层依赖当前层时，才会真实的删除当前层。

除了镜像依赖以外，还需要注意的是容器对镜像的依赖。如果有用这个镜像启动的容器存在(即使容器没有运行)，那么同样不可以删除这个镜像。之前讲过，容器是以镜像为基础，再加一层容器存储层，组成这样的多层存储结构去运行的。因此该镜像如果被这个容器所依赖的，那么删除必然会导致故障。如果这些容器是不需
要的，应该先将它们删除，然后再来删除镜像。

- 批量删除镜像

可以使用 docker images -q 来配合使用 docker rmi ，这样可以成批的删除希望删除的镜像。

例子：

```
//删除悬空镜像
docker rmi $(docker images -q -f dangling=true)

//删除所有仓库名为 redis 的镜像
docker rmi $(docker images -q redis)

//删除所有在 mongo:3.2 之前的镜像
docker rmi $(docker images -q -f before=mongo:3.2)

```

## 9. 操作docker容器

- docker run启动容器

当利用 docker run 来创建容器时，Docker 在后台运行的标准操作包括:

a. 检查本地是否存在指定的镜像，不存在就从公有仓库下载
b. 利用镜像创建并启动一个容器 
c. 分配一个文件系统，并在只读的镜像层外面挂载一层可读写层 
d. 从宿主主机配置的网桥接口中桥接一个虚拟接口到容器中去 
e. 从地址池配置一个 ip 地址给容器
f. 执行用户指定的应用程序
g. 执行完毕后容器被终止

例如：` sudo docker run -t -i ubuntu:14.04 /bin/bash`

其中， -t 选项让Docker分配一个伪终端(pseudo-tty)并绑定到容器的标准输入 上，   -i 则让容器的标准输入保持打开。

加一个-d参数后台运行,例如; `sudo docker run -d ubuntu:14.04 `

- docker logs查看容器日志

格式：`docker logs [container ID or NAMES]`

- docker start

启动已经停止的容器

- docker stop

停止容器

- docker restart

重启容器

- 导出容器

![](https://raw.githubusercontent.com/heyuan110/static-source/master/media/15575697047773/15579916274906.jpg)

- 导入容器快照

可以使用 docker import 从容器快照文件中再导入为镜像，例如

```
$ cat ubuntu.tar | sudo docker import - test/ubuntu:v1.0
$ sudo docker images
REPOSITORY          TAG                 IMAGE ID            CREA
TED              VIRTUAL SIZE
test/ubuntu         v1.0                9d37a6082e97        Abou
t a minute ago   171.3 MB
```

注:用户既可以使用 docker load 来导入镜像存储文件到本地镜像库，也可以 使用 docker import来导入一个容器快照到本地镜像库。这两者的区别在于容 器快照文件将丢弃所有的历史记录和元数据信息(即仅保存容器当时的快照状 态)，而镜像存储文件将保存完整记录，体积也要大。此外，从容器快照文件导入 时可以重新指定标签等元数据信息。

- 删除容器

docker rm 

- 清理所有处于终止状态的容器

`docker rm $(docker ps -a -q)`

## 10. 仓库

- 官方仓库[docker hub](https://hub.docker.com/)

docker search xx
docker pull xx

- 私有仓库

创建私有仓库`docker run --name=myregistry -d -p 8085:5000 -v /usr/local/programs/docker/myregistry:/var/lib registry`

标记镜像`docker tag`，然后推送到私有仓库,例如：

```
//打tag，xxxxxx是镜像id或名称
docker tag xxxxxx 172.16.166.130:8085/test_nginx

//推送到私有仓库
docker push 172.16.166.130:8085/test_nginx

```

查看私有仓库镜像: `curl http://172.16.166.130:8085/v2/_catalog`

到一台新机器pull私有仓库镜像：`docker pull 172.16.166.130:8085/test_nginx`，
注意在pull前请将http://172.16.166.130:8085地址添加到docker配置文件里。ubuntu配置(/etc/docker/daemon.json)格式：

```
{
    "insecure-registries": [
        "172.16.166.130:8085"
    ]
}
```

## 11. 数据卷

数据卷是一个可供一个或多个容器使用的特殊目录，它绕过 UFS，可以提供很多有 用的特性:
 
    - 数据卷可以在容器之间共享和重用
    - 对数据卷的修改会立马生效
    - 对数据卷的更新，不会影响镜像
    - 数据卷默认会一直存在，即使容器被删除

- 创建数据卷

在用 docker run 命令的时候，使用 -v 标记来创建一个数据卷并挂载到容器 里。在一次 run 中多次使用可以挂载多个数据卷。例如 `docker run -d -P --name web -v /webapp training/webapp`

也可以在 Dockerfile 中使用 VOLUME 来添加一个或者多个新的卷到由该镜 像创建的任意容器。

- 挂载一个主机目录作为数据卷

使用 -v 标记也可以指定挂载一个本地主机的目录到容器中去, 例如：

`docker run -d -P --name web -v /src/webapp:/opt/webapp training/webapp python app.py`

上面的命令加载主机的 /src/webapp 目录到容器的 /opt/webapp 目录, 
主机本地目录的路径必须是绝对路径，如果目录不存在 Docker 会自动为你创建它。
注意:Dockerfile 中不支持这种用法，这是因为 Dockerfile 是为了移植和分享用 的。然而，不同操作系统的路径格式不一样，所以目前还不能支持。

Docker 挂载数据卷的默认权限是读写，用户也可以通过 :ro 指定为只读。

`docker run -d -P --name web -v /src/webapp:/opt/webapp:ro
training/webapp python app.py`

加了 :ro 之后，就挂载为只读了。

- 查看数据卷的具体信息

`docker inspect web`

- 挂载一个本地主机文件作为数据卷

-v 标记也可以从主机挂载单个文件到容器中,例如：

`docker run --rm -it -v ~/.bash_history:/.bash_history ubuntu /bin/bash`

这样就可以记录在容器输入过的命令了。注意:如果直接挂载一个文件，很多文件编辑工具，包括 vi 或者 sed --in- place ，可能会造成文件 inode 的改变，从 Docker 1.1 .0起，这会导致报错误信 息。所以最简单的办法就直接挂载文件的父目录。


## 12. 数据卷容器

如果你有一些持续更新的数据需要在容器之间共享，最好创建数据卷容器。 数据卷容器，其实就是一个正常的容器，专门用来提供数据卷供其它容器挂载的。

使用`--volumes-from`挂载容器中的数据卷，例如：

先创建一个数据卷容器：

`docker run -d -v /dbdata --name dbdata training/postgres`

然后把dbdata容器挂载到另外一个容器:

`docker run -d --volumes-from dbdata --name db1 training/p
ostgres`

如果删除了挂载的容器(包括 dbdata、db1 和 db2)，数据卷并不会被自动删除。 如果要删除一个数据卷，必须在删除最后一个还挂载着它的容器时使用 docker rm -v 命令来指定同时删除关联的容器。 

- 备份

首先使用 --volumes-from 标记来创建一个加载 dbdata 容器卷的容器，并从主 机挂载当前目录到容器的 /backup 目录。命令如下:

`docker run --volumes-from dbdata -v $(pwd):/backup ubuntu
 tar cvf /backup/backup.tar /dbdata`

容器启动后，使用了 tar 命令来将 dbdata 卷备份为容器中 /backup/backup.tar 文件，也就是主机当前目录下的名为 backup.tar 的文件。

- 恢复

如果要恢复数据到一个容器，首先创建一个带有空数据卷的容器 dbdata2。

`docker run --volumes-from dbdata2 busybox /bin/ls /dbdata`

## 13. 网络

### 13.1 映射端口

容器中可以运行一些网络应用，要让外部也可以访问这些应用，可以通过 -P 或
-p 参数来指定端口映射。

当使用 -P 标记时，Docker 会随机映射一个 49000~49900 的端口到内部容器开
放的网络端口。

例如: `docker run -d -P nginx`

当使用-p标记时，要指定主机端口和容器端口

例如: `docker run -d -p 8081:80 nginx`

将容器的80端口映射到主机的8081端口

使用`docker logs -f 容器名`可以查看容器日志.

- 映射到指定地址的指定端口

格式：`ip:hostPort:containerPort`，例如:`docker run -d -p 127.0.0.1:5000:5000 training/webapp python app.py`

- 映射到指定地址的任意端口

格式：ip::containerPort，例如绑定 localhost 的任意端口到容器的 5000 端口，本地 主机会自动分配一个端口。

`run -d -p 127.0.0.1::5000 training/webapp python app.py`

还可以使用 udp 标记来指定 udp 端口：`docker run -d -p 127.0.0.1:5000:5000/udp training/webapp python app.py`

- 查看映射端口配置

`docker port 容器名`

注意: 
a. 容器有自己的内部网络和 ip 地址(使用 docker inspect 可以获取所有的 变量，Docker 还可以有一个可变的网络配置。)
b. -p 标记可以多次使用来绑定多个端口

例如：`docker run -d -p 5000:5000  -p 3000:80 training/webapp py
thon app.py`

- 容器互联

容器的连接(linking)系统是除了端口映射外，另一种跟容器中应用交互的方式。
该系统会在源和接收容器之间创建一个隧道，接收容器可以看到源容器指定的信
息。

是用--name参数命名容器，例如: `docker run -d -P --name web training/webapp`

可以用`docker ps -l`或`docker inspect`查看容器名称

注意:容器的名称是唯一的。如果已经命名了一个叫 web 的容器，当你要再次使用 web 这个名称的时候，需要先用 docker rm 来删除之前创建的同名容器。

在执行 的时候如果添加 --rm 标记，则容器在终止后会立刻删 除。注意,--rm和-d参数不能同时使用。

使用 --link 参数可以让容器之间安全的进行交互。--link参数的格式为 --link name:alias ，其中 name 是要链接的容器的 名称，alias是这个连接的别名。例如：

首先创建一个数据库容器: `docker run -d --name db training/postgres`
再创建一个web容器：`docker run -d -P --name web --link db:db training/webapp
python app.py`

这样web和db两个容器就关联了。

Docker 在两个互联的容器之间创建了一个安全隧道，而且不用映射它们的端口到 宿主主机上。在启动 db 容器的时候并没有使用 -p 和 -P 标记，从而避免了暴 露数据库端口到外部网络上。

Docker 通过 2 种方式为容器公开连接信息: 环境变量和更新 /etc/hosts 文件 ，例如

查看一下web2关联db后的环境变量,执行`docker run --rm --name web2 --link db:db training/webapp env` 结果:

```
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
HOSTNAME=8aa5fc9a8c74
DB_PORT=tcp://172.17.0.9:5432
DB_PORT_5432_TCP=tcp://172.17.0.9:5432
DB_PORT_5432_TCP_ADDR=172.17.0.9
DB_PORT_5432_TCP_PORT=5432
DB_PORT_5432_TCP_PROTO=tcp
DB_NAME=/web2/db
DB_ENV_PG_VERSION=9.3
HOME=/root
```

其中 DB_ 开头的环境变量是供 web 容器连接 db 容器使用，前缀采用大写的连接 别名。

查看一下web2的/etc/hosts文件，执行`docker run -t -i --rm --link db:db training/webapp /bin/bash`，进入后执行`cat /etc/hosts`结果：

```
127.0.0.1	localhost
::1	localhost ip6-localhost ip6-loopback
fe00::0	ip6-localnet
ff00::0	ip6-mcastprefix
ff02::1	ip6-allnodes
ff02::2	ip6-allrouters
172.17.0.9	db 79e00cc81a64
172.17.0.11	9edb693775f6
```

域名db解析到了172.17.0.9，我们执行下ping命令：

```
root@9edb693775f6:/opt/webapp# ping db
PING db (172.17.0.9) 56(84) bytes of data.
64 bytes from db (172.17.0.9): icmp_seq=1 ttl=64 time=0.195 ms
64 bytes from db (172.17.0.9): icmp_seq=2 ttl=64 time=0.077 ms
64 bytes from db (172.17.0.9): icmp_seq=3 ttl=64 time=0.097 ms
64 bytes from db (172.17.0.9): icmp_seq=4 ttl=64 time=0.064 ms
64 bytes from db (172.17.0.9): icmp_seq=5 ttl=64 time=0.059 ms
64 bytes from db (172.17.0.9): icmp_seq=6 ttl=64 time=0.078 ms
64 bytes from db (172.17.0.9): icmp_seq=7 ttl=64 time=0.084 ms
64 bytes from db (172.17.0.9): icmp_seq=8 ttl=64 time=0.089 ms
64 bytes from db (172.17.0.9): icmp_seq=9 ttl=64 time=0.074 ms
64 bytes from db (172.17.0.9): icmp_seq=10 ttl=64 time=0.074 ms
64 bytes from db (172.17.0.9): icmp_seq=11 ttl=64 time=0.059 ms
64 bytes from db (172.17.0.9): icmp_seq=12 ttl=64 time=0.098 ms
64 bytes from db (172.17.0.9): icmp_seq=13 ttl=64 time=0.061 ms
64 bytes from db (172.17.0.9): icmp_seq=14 ttl=64 time=0.074 ms
```

用 ping 来测试db容器，它会解析成 172.17.0.5 。 

*注意:官方的 ubuntu 镜像 默认没有安装 ping，需要自行安装。*

### 13.2 高级网络配置

当 Docker 启动时，会自动在主机上创建一个 docker0 虚拟网桥，实际上是 Linux 的一个 bridge，可以理解为一个软件交换机。它会在挂载到它的网口之间进 行转发。

![](https://raw.githubusercontent.com/heyuan110/static-source/master/media/15575697047773/15580787948502.jpg)


![](https://raw.githubusercontent.com/heyuan110/static-source/master/media/15575697047773/15580788457994.jpg)


从网络架构的角度来看，所有的容器通过本地主机的网桥接口相互通信，就像物理机器通过物理交换机通信一样。

一些要熟悉的概念：

![](https://raw.githubusercontent.com/heyuan110/static-source/master/media/15575697047773/15580795828187.jpg)

## 14. Docker基本架构

 ![](https://raw.githubusercontent.com/heyuan110/static-source/master/media/15575697047773/15580802323092.jpg)

Docker 采用了 C/S架构，包括客户端和服务端。 Docker daemon 作为服务端接受 来自客户的请求，并处理这些请求(创建、运行、分发容器)。 客户端和服务端既 可以运行在一个机器上，也可通过 socket 或者 RESTful API 来进行通信。






