---
title: Jenkins + AWS CodeDeploy + AutoScaling 持续集成
date: 2018-11-20 11:05:40
toc: true
tags:
    - Jenkins
    - CodeDeploy
    - AutoScaling
    - 蓝绿部署
    - 持续集成
    - 灰度发布
---

本文主要记录如何结合jenkins,codedeploy,s3, autoscaling等相关服务搭建一套可持续交付和应用部署的服务。

## Aws AutoScaling部分

### 1、使用Auto Scaling的优点

1)、保持基础设置堆栈配置一致(例如软件nginx、php等安装配置一致)

2)、快速设置扩展(只需要设定Auto Scaling组内的所需实例数量即可完成实例的扩展)

3)、制定明确的扩展策略(比如根据CPU的利用率时增加或是缩减实例数量)

4)、控制实例资源成本(在Auto Scaling组内的实例通常都比较小，不然就失去了AutoScaling自动扩展的意义)

### 2、Auto Scaling组件说明

Auto Scaling组：EC2实例放在组中，用于扩展和管理的逻辑单元，在创建组时，可以指定其最小，最大和所需EC2的实例数量

(备注：auto Scaling 组是与启动配置相关联的)

Auto Scaling启动配置：EC2实例启动时的模板(指定实例类型，密钥对，安全组和磁盘空间大小)

### 3、创建启动配置步骤

1)、为EC2实例创建IAM角色

https://console.aws.amazon.com/iam/

![](https://raw.githubusercontent.com/heyuan110/static-source/master/media/15426810514625/15426812146538.jpg)

![](https://raw.githubusercontent.com/heyuan110/static-source/master/media/15426810514625/15426812429125.jpg)

![](https://raw.githubusercontent.com/heyuan110/static-source/master/media/15426810514625/15426812509358.jpg)
![](https://raw.githubusercontent.com/heyuan110/static-source/master/media/15426810514625/15426812612000.jpg)

2)、为Auto Scaling 创建启动配置(下图为已经设置好的启动配置模板)

![](media/15426810514625/15426813227709.jpg)

3)、为EC2实例启动时添加必要的用户数据(主要为应用程序事先搭建好基础环境)

![](https://raw.githubusercontent.com/heyuan110/static-source/master/media/15426810514625/15426814249267.jpg)

脚本如下：

```
#!/bin/bash

#run all command as root,system: ubuntu16.04

#enter /opt/patpat-devops folder

cd /opt/patpat-devops

#download aws codedeploy agent,and start

rm -rf codedeploy-agent

mkdir codedeploy-agent

cd codedeploy-agent

wget https://xxxx.com/latest/install

chmod +x ./install

./install auto

service codedeploy-agent start

service codedeploy-agent status

#backto patpat-devops folder

cd /opt/patpat-devops

#download nginx1.14, not need single start

wget https://xxxx.com/nginx/nginx1.14.tar

tar -xf nginx1.14.tar

mv nginx /usr/local/programs/

#replace /etc/logrotate.d/nginx

\cp -f /usr/local/programs/nginx/logrotate.d/nginx /etc/logrotate.d/nginx

#copy supervisor.d/nginx.conf to supervisor

\cp -f /usr/local/programs/nginx/supervisor.d/nginx.conf /usr/local/programs/supervisor/conf.d/

rm nginx1.14.tar

#download php7,not need single start

wget https://xxxx.com/php/php7.tar

tar -xf php7.tar

mv php7 /usr/local/programs/

#copy supervisor.d/php-fpm.conf to supervisor

\cp -f /usr/local/programs/php7/etc/supervisor.d/php-fpm.conf /usr/local/programs/supervisor/conf.d/

rm php7.tar

#start supervisor,nginx,php,zabbix start server

isExistApp=`pgrep supervisor`

if [ -n  $isExistApp ]; then

    echo "supervisor is activing, will reload"

    #restart server

    /usr/local/bin/supervisorctl reload 

else

    echo "supervisor is not active, will start server"

    #start server

   /usr/local/bin/supervisord -c /usr/local/programs/supervisor/supervisord.conf     

fi

#change nginx log permession

chmod -R 777 /usr/local/programs/nginx/logs

echo "Install finished!"

```

备注：具体在创建实例的时候在"配置详细信息"选项卡里面进行添加

![](https://raw.githubusercontent.com/heyuan110/static-source/master/media/15426810514625/15426815907986.jpg)

4)、创建Auto Scaling 组 (下图为已经创建好的Auto Scaling组)

![](https://raw.githubusercontent.com/heyuan110/static-source/master/media/15426810514625/15426816027339.jpg)

Auto Scaling结合ELB使用

![](https://raw.githubusercontent.com/heyuan110/static-source/master/media/15426810514625/15426816334272.jpg)

此时aws auto scaling创建完成，随后Auto Scaling会根据给定的按需实例设置启动一台EC2实例

## AWS Code Deploy部分

### 1、CodeDeploy部署的前提条件

需要在Auto Scaling启动配置中的EC2启动模板事先安装好CodeDeploy agent代理
Install or reinstall the aws CodeDeploy agent for Amazon Ubuntu Server

```

$ sudo apt-get install ruby

$ sudo apt-get install wget

$ cd /home/ubuntu

$ wget https://bucket-name.s3.amazonaws.com/latest/install

$ chmod +x ./install

$ sudo ./install auto

$ sudo service codedeploy-agent status

$ sudo service codedeploy-agent start

$ sudo service codedeploy-agent status

```

备注：bucket-name 是包含适用于您所在区域的AWS CodeDeploy 资源工具包文件的 Amazon S3sds-s3-latest-bucket-name 存储桶的名称，对于美国东部(俄亥俄)区域，bucket-name 替换为 aws-codedeploy-us-east-2

### 2、Code Deploy组件

**部署**：部署一个包括应用程序和AppSpec文件的新修订；AppSpec指定如何将应用程序部署到部署组中的实例

**应用程序**：部署组和修订的集合；EC2/本地应用程序使用EC2/本地计算平台

**修订版**：修订版是可部署内容(如源代码、构建后项目、网页、可执行文件和部署脚本以及 AppSpec 文件)的特定版本，AWS CodeDeploy 代理程序可通过 GitHub 或 Amazon S3 存储桶访问修订版

**部署组**：部署组是用于在CodeDeploy部署中对EC2实例的AWS CodeDeploy实体；它是一组与您以之为目标进行部署的应用程序关联的实例；可以通过指定标签、Auto Scaling组名称或同时指定此二者，将实例添加到部署组中

备注：可以为一个应用程序定义多个部署组，如模拟和生产

**部署配置**：部署配置为部署组指定如何进行部署的行为，包括如何处理部署故障；可以使用部署配置向多实例部署组执行零停机部署；例如，如果您的应用程序需要部署组中至少有50% 的实例在运行中且提供流量，可以在您的部署配置中指定这一点，从而使部署不会导致停机；如果没有与部署或者部署组相关联的部署配置，则在默认情况下，AWS CodeDeploy将会一次部署到一个实例中

### 3、AWS CodeDeploy部署类型

**就地部署**

停止部署组中每个实例上的应用程序，安装最新的应用程序修订版，然后启动和验证应用程序的新版本；您可以使用负载均衡器，以便在部署期间取消注册每个实例，然后在部署完成后让其重新提供服务；只有使用EC2/本地计算平台的部署才能使用就地部署

对部署组中的实例依次执行脱机操作/更新应用/恢复联机的操作，完成滚动部署

**蓝绿部署**

创建一组新的替换实例，并安装新版本的应用程序。成功后，切换流量到这些新实例，删除旧实例，完成部署；AWS CodeDeploy 运行您在切换流量之前，对新版本应用程序进行测试；如果发现问题，您可以快速回滚到旧版本

EC2/本地计算平台上的蓝/绿部署：部署组中的实例(原始环境)将被不同的实例集(替代环境)所代替，步骤如下：

1)、系统将为替代环境配置实例

2)、替代实例上将安装最新的应用程序修订

3)、对于应用程序测试和系统验证等活动来说，等待时间可选

4)、替代环境中的实例在Elastic Load Balancing负载均衡器中进行注册，使得流量重新路由至这些实例；系统将撤销原始环境中的实例注册，进而终止或因其他使用情形而保持运行

备注：蓝/绿部署只能与Amazon EC2实例配合使用

### 4、就地部署概述

停止部署组中每个实例上的应用程序，安装最新的应用程序修订版，然后启动和验证应用程序的新版本。您可以使用负载均衡器，以便在部署期间取消注册每个实例，然后在部署完成后让其重新提供服务

![](https://raw.githubusercontent.com/heyuan110/static-source/master/media/15426810514625/15426818126546.jpg)

1)、在本地开发计算机或类似环境上创建可部署的内容，然后添加application specification file (AppSpec file)；AppSpec file对AWS CodeDeploy是唯一的；它定义了AWS CodeDeploy执行的部署操作；将可部署的内容和AppSpec file捆绑成一个存档文件，然后将其上传到Amazon S3存储桶或GitHub存储库；此存档文件称为应用程序修订(简称修订)

2)、向AWS CodeDeploy提供有关您的部署的信息，例如，要从中提取修订的Amazon S3存储桶或GitHub，以及要将其内容部署到的一组Amazon EC2实例；AWS CodeDeploy将一组Amazon EC2实例称为一个部署组，部署组中包含单独标记的Amazon EC2实例和/或Amazon EC2 Auto Scaling组中的Amazon EC2实例

每次您成功上传要部署到部署组的新应用程序修订时，该捆绑包就会设置为部署组的目标修订；也就是说，当前设为部署目标的应用程序修订为目标修订，这也是将为自动部署提取的修订

3)、每个实例上的CodeDeploy-Agent将轮询 AWS CodeDeploy，以确定从指定的Amazon S3存储桶或GitHub存储库中提取的内容和提取时间

4)、每个实例上的CodeDeploy-Agent将从指定的Amazon S3存储桶或GitHub存储库中提取目标修订，并按照 AppSpec file中的说明向实例部署内容

AWS CodeDeploy将保留您的部署的记录，以便您可以获取部署状态、部署配置参数、实例运行状况

### 5、蓝/绿部署概述

*在蓝/绿部署中将流量从一组实例(原始环境)重新路由到另一组实例(替换环境)*，相比就地部署提供了多种优势：

1)、可以提前在新实例上安装和测试应用程序，并通过直接将流量切换到新服务器来将应用程序部署到生产中

2)、可以更快、更可靠地切换回最近的应用程序版本，因为只要原始实例没有终止，就可以将流量路由回原始实例；而在就地部署中，必须通过重新部署上一个版本的应用程序来回滚版本

3)、如果您使用EC2/本地计算平台，则会为蓝/绿部署预置新实例，并且新实例反映最新的服务器配置；这将帮助您避免在长时间运行的实例上有时出现的问题类型

wiki：https://docs.aws.amazon.com/zh_cn/codedeploy/latest/userguide/welcome.html#welcome-deployment-overview-in-place

### 6、AppSpec 文件

appspec.yml是YAML格式、用于定于CodeDeploy服务在整个阶段所做的操作和文件拷贝路径和权限等。这个文档名称必须是appspec.yml,而且文档中的空格个数也有严格的要求,[参考详细解析
](https://blog.csdn.net/fedora18/article/details/44237647). 

如果没有AppSpec file，AWS CodeDeploy无法将应用程序修订中的源文件映射到其目标，也无法为您向EC2/本地计算平台中进行的部署运行脚本

AppSpec 文件是一种配置文件，用于指定待复制文件和待执行脚本；AppSpec文件使用 YAML 格式，它位于您的修订版的根目录下；AppSpec文件为AWS CodeDeploy 代理程序所用，由两个部分组成；文件部分指定了您的版次中待复制的源文件，以及每个实例的目标文件夹；挂接部分指定了在部署各阶段运行的脚本的位置(作为从修订包根下起始的相对路径)；部署的各阶段被称为部署生命周期事件；下面是一个示例 AppSpec文件

```
appspec.yml示例：

version: 0.0                                      # 固定写法

os: linux

files:                                                 # You can specify one or more mappings in the files section.

  - source: /

    destination: /var/www/html/WordPress

hooks:                                              # The lifecycle hooks sections allows you to specify deployment scripts.

ApplicationStop:                             # Step 1: Stop Apache and MySQL if running.

    - location: helper_scripts/stop_server.sh

BeforeInstall:                                  # Step 2: Install Apache and MySQL.

# You can specify one or more scripts per deployment lifecycle event.

    - location: deploy_hooks/puppet-apply-apache.sh

    - location: deploy_hooks/puppet-apply-mysql.sh

AfterInstall:

    - location: deploy_hooks /change_permissions.sh              # Step 3: Set permissions.

      timeout: 30

      runas: root

    - location: helper_scripts/start_server.sh                             # Step 4: Start the server.

      timeout: 30

      runas: root

```

appspec.yml生产环境示例：

```
version: 0.0                                        # AppSpec file的版本；允许的唯一值为0.0

os: linux                                             # 指定部署到的实例的操作系统值(linux、windows)

files:                                                   # 定义文件映射关系

  - source: /                                        # 表示本版本包的全部文件和目录(相对路径，从修订的根目录开始)

    destination: /var/www/alpha-website_desktop/       # 被部署服务器的完整路径

hooks:                                               # 定义CodeDeploy各阶段执行的操作

  BeforeInstall:                                  # CodeDeploy部署之前

    - location: codedeploy-scripts/install_dependencies    # 运行的软件包依赖脚本

      timeout: 300                               # 安装超时时间

      runas: root                                 # 以什么用户进行软件安装

  AfterInstall:                                    # CodeDeploy部署之后

    - location: codedeploy-scripts/change_applications_configure

    - location: codedeploy-scripts/change_permissions

      timeout: 300

      runas: root

  ApplicationStart:                           # 应用启动

    - location: codedeploy-scripts/start_server

      timeout: 300

      runas: root

  ApplicationStop:                           # 应用停止

    - location: codedeploy-scripts/stop_server

      timeout: 300

      runas: root
```

appspec.yml部署生命周期：

![](https://raw.githubusercontent.com/heyuan110/static-source/master/media/15426810514625/15426821571744.jpg)

部署生命周期说明

部署会经过一组预定义阶段，称为部署生命周期事件。部署生命周期事件可让您将代码作为部署的一部分运行

下表以执行顺序列出了目前支持的各种不同的部署生命周期事件，以及您可能想使用它们的时间示例
![](https://raw.githubusercontent.com/heyuan110/static-source/master/media/15426810514625/15426821846445.jpg)

In-place deployments(就地部署)
![](https://raw.githubusercontent.com/heyuan110/static-source/master/media/15426810514625/15426822011681.jpg)

Blue/green deployments(蓝绿部署生命周期)

![](https://raw.githubusercontent.com/heyuan110/static-source/master/media/15426810514625/15426822146656.jpg)

蓝绿部署流量切换过程

![](https://raw.githubusercontent.com/heyuan110/static-source/master/media/15426810514625/15426822261108.jpg)

CodeDeploy部署方式：

![](https://raw.githubusercontent.com/heyuan110/static-source/master/media/15426810514625/15426822356524.jpg)

CodeDeploy与AutoScaling集成原理 https://aws.amazon.com/cn/blogs/devops/under-the-hood-aws-codedeploy-and-auto-scaling-integration/

## 使用Auto Scaling配置CodeDeploy

使用Auto Scaling配置CodeDeploy非常简单。只需转到AWS CodeDeploy控制台，然后在部署组配置中指定Auto Scaling组名称

![](https://raw.githubusercontent.com/heyuan110/static-source/master/media/15426810514625/15426823243690.jpg)

此外，还需要：

1、在Auto Scaling实例上安装CodeDeploy代理。您可以将代理程序作为基本AMI的一部分进行烘焙，也可以在启动期间使用用户数据来安装代理程序

2、确保CodeDeploy用于与Auto Scaling交互的服务角色具有正确的权限

### Auto Scaling Lifecycle Hook

事件中的Auto Scaling和CodeDeploy之间的通信基于Auto Scaling生命周期挂钩；建议不要尝试手动设置或修改这些挂钩，因为CodeDeploy可以为您执行此操作；Auto Scaling生命周期挂钩告诉Auto Scaling在实例即将更改为某些Auto Scaling生命周期状态时发送通知；CodeDeploy仅侦听有关已启动且即将放入InService的实例的通知；此状态发生在EC2实例完成引导之后，但在它被放置到您已配置的任何Elastic Load Balancing负载平衡器之后；Auto Scaling在继续处理实例之前等待CodeDeploy的成功响应

![](https://raw.githubusercontent.com/heyuan110/static-source/master/media/15426810514625/15426823485910.jpg)

挂钩是Auto Scaling组配置的一部分；您可以使用describe-lifecycle-hooks CLI命令查看Auto Scaling组上安装的挂钩列表；创建或修改部署组以包含Auto Scaling组时，CodeDeploy将执行以下操作：

1)、使用传入的CodeDeploy服务角色与部署组一起使用以获取Auto Scaling组的权限

2)、在Auto Scaling组中安装生命周期挂钩，用于实例启动，将通知发送到CodeDeploy拥有的队列

3)、将已安装的挂钩的记录添加到部署组

从部署组中删除Auto Scaling组或删除部署组时，CodeDeploy将执行以下操作：

1)、使用部署组的服务角色来访问Auto Scaling组

2)、获取部署组中记录的挂钩，并将其从Auto Scaling挂钩中删除

3)、如果正在修改(未删除)部署组，请从部署组中删除该挂钩的记录

如果创建钩子有问题，CodeDeploy将尝试回滚更改；如果删除钩子有问题，CodeDeploy将在API响应中返回不成功的钩子删除并继续

以下是Auto Scaling scale-in事件期间发生的事件序列：

1、Auto Scaling要求EC2提供新实例

2、EC2使用Auto Scaling提供的配置旋转新实例

3、Auto Scaling查看新实例，将其置于Pending：Wait状态，并将通知发送到Code Deploy

4、CodeDeploy从Auto Scaling接收实例启动通知

5、CodeDeploy验证实例和部署组的配置

1)、如果通知看起来正确，但部署组不再包含Auto Scaling组(或者我们可以确定先前已删除部署组)，则CodeDeploy将不会部署任何内容，并通过实例启动将Auto Scaling告知CONTINUE；Auto Scaling将尊重实例启动时的任何其他约束; 如果出现其他问题，此步骤不会强制Auto Scaling继续

2)、如果CodeDeploy无法处理消息(例如，如果存储的服务角色未授予适当的权限)，则CodeDeploy将使挂钩超时；CodeDeploy的默认超时为10分钟

6、CodeDeploy为实例创建新部署以部署部署组的目标修订；(目标修订版是部署组的最后一个成功部署的修订版；它由CodeDeploy维护)；您需要至少部署到部署组一次，以便CodeDeploy识别目标修订版；您可以使用get-deployment-group CLI命令或CodeDeploy控制台获取部署组的目标修订

1)、部署正在运行时，它会将心跳发送到Auto Scaling，让它知道该实例仍在处理中

2)、如果部署出现问题，CodeDeploy将立即告知Auto Scaling ABANDON实例启动。Auto Scaling终止实例并使用新实例重新启动该过程

![](https://raw.githubusercontent.com/heyuan110/static-source/master/media/15426810514625/15426823677871.jpg)

最佳实践
1)、设置或修改Auto Scaling生命周期挂钩 -不要手动设置或修改Auto Scaling挂钩，因为配置错误可能会破坏CodeDeploy集成(备注：在CodeDeploy中配置添加AutoScaling组，生命周期挂钩就已经建立)

2)、注意部署失败 - 当部署到新实例失败时，CodeDeploy会将实例标记为终止；Auto Scaling将终止实例，启动新实例，并通知CodeDeploy以启动部署；当你有暂时的错误时，这很好。但是，缺点是如果您的目标修订版存在问题(例如，如果部署脚本中存在错误)，则启动和终止实例的此循环可以进入循环；我们建议您密切监视部署并设置Auto Scaling 通知，以跟踪Auto Scaling启动和终止的EC2实例

3)、Auto Scaling部署疑难解答-对涉及Auto Scaling组的部署进行故障排除可能具有挑战性；如果部署失败，我们建议您取消Auto Scaling组与部署组的关联，以防止Auto Scaling不断启动和终止EC2实例。接下来，将使用相同基本AMI启动的标记EC2实例添加到部署组，将目标修订部署到该EC2实例，并使用该实例对脚本进行故障排除。如果您有信心，请将部署组与Auto Scaling组关联，将黄金版本部署到Auto Scaling组，扩展新的EC2实例（通过调整Min，Max和Desired（）），并验证部署是否成功

4)、订购启动脚本的执行 - CodeDeploy代理在启动时立即查找并执行部署；部署执行和启动脚本(如用户数据，cfn-init等)之间没有排序；我们建议您将启动脚本作为启动脚本的一部分(也可能是最后一步)安装，以便您可以确定在实例安装了不属于CodeDeploy部署的依赖项之前，不会执行部署；如果您希望将代理程序烘焙到基本AMI中，我们建议您将代理程序服务保持在停止状态，并使用启动脚本启动代理程序服务

5)、将多个部署组与同一Auto Scaling组关联 - 通常，应避免将多个部署组与同一Auto Scaling组关联；当Auto Scaling使用与多个部署组关联的多个挂钩扩展实例时，它会同时发送所有挂钩的通知；因此，将创建多个CodeDeploy部署；这有几个缺点。这些部署是并行执行的，因此您将无法依赖它们之间的任何顺序；如果任何部署失败，Auto Scaling将立即终止实例；当实例关闭时，正在运行的其他部署将开始失败，但它们可能需要一个小时才能完成；Host Agent一次只处理一个部署命令，因此您还需要考虑两个限制；第一，其中一个部署可能会因为时间过长而失败；例如，如果部署中的步骤需要五分钟以上才能完成，则可能会发生这种情况；其次，部署之间没有先发制人，因此无法在一个部署与另一个部署之间强制执行步骤排序；因此，我们建议您最小化与Auto Scaling组关联的部署组的数量，并将部署合并到单个部署中

## Jenkins+Git+CodeDeploy持续集成

参考官方教程: https://aws.amazon.com/cn/blogs/china/aws-devops-jenkins-and-codedeploy/

需要Jenkins安装codedeploy插件，插件截图：

![](https://raw.githubusercontent.com/heyuan110/static-source/master/media/15426810514625/15426825748300.jpg)


### Jenkins关于项目配置目录规则

```
project-configure/

├── geodata

│   └── geoip.mmdb

├── xxxx                                                                               # 公司名称

│   ├── alpha                                                                           # alpha环境

│   │   ├── xxxx-alpha-website_api                                    # 具体的项目名称

│   │   │   ├── codedeploy                                                      # CodeDeploy目录存放codedeploy必须的appspec.yml和脚本文件夹

│   │   │   │   ├── appspec.yml                                               # 用于定义CodeDeploy服务在整个阶段做的操作和文件拷贝路径和权限等

│   │   │   │   └── codedeploy-scripts                                   # CodeDeploy中脚本执行所在文件夹

│   │   │   │       ├── change_applications_configure          # CodeDeploy部署完成后，配置文件拷贝，服务重启操作

│   │   │   │       ├── change_permissions                            # codedeploy部署完成后，相关目录权限变更等设置

│   │   │   │       ├── install_dependencies                           # codedeploy部署前系统软件包依赖关系安装

│   │   │   │       ├── start_server                                           # 启动服务脚本

│   │   │   │       └── stop_server                                           # 停止服务脚本

│   │   │   ├── project-conf.d                                                # 项目中的.env配置文件目录

│   │   │   └── server-applications                                       # 软件(nginx，php)应用程序配置文件存放文件夹

│   │   │       └── nginx

│   │   │           └── conf.d

│   │   │               └── xxxx-alpha-website_api.conf            

│   ├── production                                                                # 线上生产环境

│   │   ├── xxxx-api

│   │   │   ├── codedeploy

│   │   │   │   ├── appspec.yml

│   │   │   │   └── codedeploy-scripts

│   │   │   │       ├── change_applications_configure

│   │   │   │       ├── change_permissions

│   │   │   │       ├── install_dependencies

│   │   │   │       ├── start_server

│   │   │   │       └── stop_server

│   │   │   ├── project-conf.d

│   │   │   └── server-applications

│   │   │       └── aws-kinesis-agent

│   │   │           └── agent.json

│   └── test                                                                           # 测试环境

└── README                                                                       # CodeDeploy目录说明
```

**相关目录说明：**

目录层级：

平台/环境/项目

目前只有patpat和ace平台，环境有production、test、alpha

项目目录包含项目配置，codedeploy配置，服务器端配置，每个项目配置下面分别有下面三个目录：

codedeploy/

project-conf.d/

server-applications/

说明：

1. codedeploy目录存放codedeploy必须的appspec.yml和scripts脚本文件

2. project-conf.d目录存放项目参数配置文件，例如.env

3. server-applications目录存放目标服务器上的nginx，php，kinesis-agent等应用程序配置，需要结合上面的codedeploy使用

务必严格按照此格式配置，每次修改配置后需要重新在jenkins上构建发布！

AWS codedeploy采取蓝绿部署，无状态部署，所以每次修改配置后需要重新构建启动新机器

### 构建截图

![](https://raw.githubusercontent.com/heyuan110/static-source/master/media/15426810514625/15426828768145.jpg)


### Jenkins与codedeploy结合部署文件上传至指定的S3存储桶

![](https://raw.githubusercontent.com/heyuan110/static-source/master/media/15426810514625/15426829980489.jpg)


![](https://raw.githubusercontent.com/heyuan110/static-source/master/media/15426810514625/15426830240171.jpg)


### 发布流程

Jenkins主要实现将构建好的部署包上传至s3存储桶，事先在EC2实例上的CodeDeploy-Agent在轮询过程中发现s3存储桶上有新的修订版(jenkins部署上传到S3的压缩包)时，获取存储桶的新的修订版并解压，CodeDeploy根据新的修订版里的appspec.yml对EC2实例进行自动部署，这样保证每次在同一个CodeDeploy组内的实例获取的都是最新的部署包.




