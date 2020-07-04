---
title: 消息队列-RabbitMQ
subtitle: 
date: 2019-08-02 11:47:04
toc: true
tags:
    - 消息队列
    - middleware
    - rabbitmq
---

前面文章[《消息队列中间件选型》](http://www.heyuan110.com/2019-07-31-%E6%B6%88%E6%81%AF%E9%98%9F%E5%88%97%E4%B8%AD%E9%97%B4%E4%BB%B6%E9%80%89%E5%9E%8B.html)我们了解了消息队列技术选型，本文我们来学习开源消息队列RabbitMQ。


## 1. RabbitMQ简介

RabbitMQ是一个开源的消息代理和队列服务器，用来通过普通协议在完全不同的应用之间共享数据，RabbitMQ是使用Erlang语言来编写的，并且RabbitMQ是基于AMQP协议的。

## 2. RabbitMQ工作流程

RabbitMQ架构图
![](https://raw.githubusercontent.com/heyuan110/static-source/master/media/rabbitmq/15647300964346.jpg)

发布者P-Clients（Publisher）发布消息(Message)，经由交换机X（Exchange）
交换机根据路由规则将收到的消息分发给与该交换机绑定的队列(Queue)
最后RabbitMQ代理（Broker）会将消息投递（Push）给订阅了此队列的消费者(Consumer)，或者消费者按照需求自行获取(Pull)

深入理解：

(1)发布者、交换机、队列、消费者都可以有多个，因为RabbitMQ是基于网络协议AMQP，所以这个过程中发布者，交换机，队列，消费者可以处于不同的设备上。

(2)消息发布者可以给消息指定各种属性（Message Meta-data),一些属性可能被消息代理(Broker)使用，一些只能被消费者使用。消息分两部分：payload(有效载荷，传输的数据)和label(标签,exchange的名字或者说是一个tag，它描述了payload)，RabbitMQ通过label决定把Message发给哪个消费者。AMQP协议仅仅描述了label，RabbitMQ决定了如何使用label的规则。消费者在收到消息时只有payload部分，label已经被删掉，对于消费者而言是不知道谁发送的消息。

(3)从安全和可靠性角度，RabbitMQ很好实现了AMQP协议的消息确认(Message Acknowledgements)机制,当一个消息投递给消费者后，不会立即删除，直到它收到来自消费者的确认回执(Acknowledgements)后,才完成从队列里删除。

## 3. RabbitMQ基本概念

我们先来看一张RabbitMQ管理界面截图

![](https://raw.githubusercontent.com/heyuan110/static-source/master/media/rabbitmq/15647344884041.jpg)

这个界面包含了RabbitMQ很关键的几个概念(不是全部)

### 3.1 Producer（生产者）

消息生产者

### 3.2 Consumer（消费者）

消息消费者

### 3.3 Connection（连接）

无论是生产者还是消费者，都需要和RabbitMQ Broker建立连接，这个连接就是一条TCP,当不要连接时，需要优雅释放掉RabbitMQ连接,而不是直接将TCP连接关闭。后面我们可以知道使用RabbitMQ程序开头时，都是先建立TCP连接。

### 3.4 Channel（信道）

一旦TCP连接建立起来，客户端紧接着可以创建一个AMQP信道（Channel），每个信道都会被指派一个唯一的ID。信道是建立在Connection之上的虚拟连接，Rabbit处理的每条指令都是通过Channel完成的。一般情况程序起始建立TCP连接，第二步就是建立Channel。

![](https://raw.githubusercontent.com/heyuan110/static-source/master/media/rabbitmq/15647378165803.jpg)

为什么要用Channel，而不直接用TCP连接？

试想如果一个场景，如果一个应用程序有很多个线程需要从RabbitMQ消费或生产消息，那么必然会创建很多个TCP连接，我们知道建立一个TCP连接需要3次握手,关闭一个TCP连接需4次挥手，对于系统来说频繁建立和关闭TCP连接对于系统性能有很大影响，而且TCP连接数也有限制，也限制了系统处理高并发的能力。
Rabbitmq采用类似NIO(也称非阻塞 I/O，包含三大核心部分：Channel信道、Buffer缓冲区和 Selector选择器)的做法,选择TCP连接复用，每个线程把持一个信道，信道复用了Connection的TCP连接.当每个信道的流量不是很大时，复用单一connection可以有效节省TCP连接资源，但如果信道流量很大，复用单一connection，connection的带宽会限制消息传输。此时需创建多个connection，将信道分摊到connection中。

### 3.5 Exchange（交换机）

消息交换机指定消息按什么规则，路由到哪个队列去。

那为什么需要Exchange而不是直接将消息发送至队列呢？

回到与RabbitM关系紧密的AMQP协议，AMQP协议核心思想就是**生产者和消费者解耦**，生产者不直接将消息发送给队列。生产者不知道消息会被发到哪个队列，它只将消息发给交换机，交换机接收到消息后按照特定的规则转发到队列进行存储。
![](https://raw.githubusercontent.com/heyuan110/static-source/master/media/rabbitmq/15647390185527.jpg)

在实际应用中我们只需要定义好Exchange的路由策略，而生产者不需要关心消息发送到哪个队列或被谁消费。消费者不需要关心谁生产，只需面向队列消费消息。

Exchange定义了消息路由到Queue的规则，将各个层面的消息传递隔离开，使每一层只需要关心自己面向的下一层，降低了整体的耦合度。

创建一个新的Exchange

![](https://raw.githubusercontent.com/heyuan110/static-source/master/media/rabbitmq/15647425737375.jpg)

- Virtual host:属于哪个Virtual host。
- Name：名字，同一个Virtual host里面的Name不能重复。
- Durability： 是否持久化，Durable：持久化。Transient：不持久化。
- Auto delete：当最后一个绑定（队列或者exchange）被unbind之后，该exchange自动被删除。
- Internal： 是否是内部专用exchange，是的话，就意味着我们不能往该exchange里面发消息。
- Arguments： 参数，是AMQP协议留给AMQP实现做扩展使用的。
- alternate_exchange配置的时候，exchange根据路由路由不到对应的队列的时候，这时候消息被路由到指定的alternate_exchange的value值配置的exchange上。

### 3.6 Exchange类型

![](https://raw.githubusercontent.com/heyuan110/static-source/master/media/rabbitmq/15647424693007.jpg)

**（1）.Direct exchange**

![](https://raw.githubusercontent.com/heyuan110/static-source/master/media/rabbitmq/15649763567956.jpg)

将消息中的Routing key与该Exchange关联的所有Binding中的Routing key进行比较，如果相等，则发送到该Binding对应的Queue中。

**（2）.Topic Exchange**
![](https://raw.githubusercontent.com/heyuan110/static-source/master/media/rabbitmq/15649764244812.jpg)

将消息中的Routing key与该Exchange关联的所有Binding中的Routing key进行对比，如果匹配上了，则发送到该Binding对应的Queue中。

**（3）.Fanout Exchange**
![](https://raw.githubusercontent.com/heyuan110/static-source/master/media/rabbitmq/15649763997958.jpg)

直接将消息转发到所有binding的对应queue中，这种exchange在路由转发的时候，忽略Routing key，所以fanout交换机也是转消息最快的。

**（4）.Headers Exchange**

将消息中的headers与该Exchange相关联的所有Binging中的参数进行匹配，如果匹配上了，则发送到该Binding对应的Queue中。

### 3.7 Queue（队列）

消息队列载体，用于存储消息，每个消息都会被投入到一个或多个队列。消息消费者就是通过订阅队列来获取消息，多个消费者可订阅同一个队列，这时队列中的消息会被平均分摊给多个消费者进行处理，而不是每个消费者都收到所有消息。

### 3.8 Binding（绑定）

绑定，它的作用就是把Exchange和Queue按照路由规则绑定起来。

### 3.9 Routing Key（路由键）

消息发送给Exchange（交换机）时，消息将拥有一个路由键（默认为空），Exchange（交换机）根据这个路由键将消息发送到匹配的队列中。

### 3.10 Binding Key（绑定键）

### 3.11 Virtual Hosts

虚拟主机，一个broker里可以开设多个vhost，用作不同用户的权限分离。

## 4. RabbitMQ队列模式

所有实例采用PHP实现，引用包[php-amqplib](https://github.com/php-amqplib/php-amqplib),引用类：

```
use PhpAmqpLib\Connection\AMQPStreamConnection;
use PhpAmqpLib\Message\AMQPMessage;
```

完整代码：<https://github.com/heyuan110/laravel-rabbitmq>

### 4.1 简单队列模式

![](https://raw.githubusercontent.com/heyuan110/static-source/master/media/rabbitmq/15649775221394.jpg)

功能：一个生产者P发送消息到队列，一个消费者C消费

生产者P代码:

```
    //https://www.rabbitmq.com/tutorials/tutorial-one-php.html
    public function publishSimpleMQ()
    {
        //create a connection
        $connection = new AMQPStreamConnection('172.16.166.130','5672','guest','guest','/');
        //create a channel
        $channel = $connection->channel();
        //declare a queue for us to send to; then we can publish a message to the queue
        $channel->queue_declare('hello',false,false,false,false);
        
        //create a message, publish it to the queue
        $msg = new AMQPMessage("hello world");
        $channel->basic_publish($msg,'','hello');
        echo " [x] Sent 'Hello World!'\n";
        Log::info(" [x] Sent 'Hello World!'\n");

        //close channel, close connection
        $channel->close();
        $connection->close();
    }
```

过程：
(1). 创建连接，设置服务地址，端口，账号，vhost
(2). 使用连接创建信道
(3). 使用信道创建队列
(4). 创建消息
(5). 使用信道向队列发送消息
(6). 关闭信道，关闭连接

消费者C代码:

```
public function consumeSimpleMQ()
    {
        //create a connection
        $connection = new AMQPStreamConnection('172.16.166.130','5672','guest','guest','/');
        //create a channel
        $channel = $connection->channel();
        //declare a queue for us to send to; then we can publish a message to the queue
        $channel->queue_declare('hello',false,false,false,false);
        
        echo " [*] Waiting for messages. To exit press CTRL+C\n";

        $callback = function ($msg){
            echo '[x] Received ' . $msg->body . '\n';
            Log::info('[x] Received ' . $msg->body . '\n');
        };
        $channel->basic_consume('hello','',false,true,false,false,$callback);
        while($channel->is_consuming()){
            $channel->wait();
        }

        //close channel, close connection
        $channel->close();
        $connection->close();
    }
```

过程：
(1). 创建连接，设置服务地址，端口，账号，vhost
(2). 使用连接创建信道
(3). 使用信道创建队列
(4). 创建消费者监听队列，从队列中读取消息
(5). 关闭信道，关闭连接

### 4.2 工作队列模式

![](https://raw.githubusercontent.com/heyuan110/static-source/master/media/rabbitmq/15649775307431.jpg)

功能：一个生产者P发送消息到队列，多个消费者C消费，每个消费者获取到的消息唯一，多个消费者只有一个队列。

工作队列：设计思路是为了避免资源密集型任务立即执行，并必须等待它完成，相反安排任务稍后执行。将任务封装成消息，后台运行的消费者进程将获取任务并最终执行，多个消费者运行时，它们之间共享任务。

生产者P代码:

```
//https://www.rabbitmq.com/tutorials/tutorial-two-php.html
 public function produceWorkMQ()
    {
        //create a connection
        $connection = new AMQPStreamConnection('172.16.166.130','5672','guest','guest','/');
        //create a channel
        $channel = $connection->channel();
        //declare a queue for us to send to; then we can publish a message to the queue
        //set queue durable is ture，make sure that messages aren't lost
        $channel->queue_declare('hello',false,$durable=true,false,false);
                
        //create a message, publish it to the queue
        $data = $this->option('msg');
        if (empty($data)) {
            $data = "Hello World!";
        }
        $msg = new AMQPMessage(
            $data,
            ['delivery_mode' => AMQPMessage::DELIVERY_MODE_PERSISTENT]
        );
        $channel->basic_publish($msg,'','hello');
        echo " [x] Sent '".$data."'\n";
        Log::info(" [x] Sent '".$data."'\n");

        //close channel, close connection
        $channel->close();
        $connection->close();
    }
```

过程：
(1). 创建连接，设置服务地址，端口，账号，vhost
(2). 使用连接创建信道
(3). 使用信道创建队列，设置队列durable参数为true，确保在rabbitmq server停止时不丢失队列
(4). 创建消息,消息内容来自输入参数msg，设置delivery_mode为2，确保在rabbitmq server停止时不丢失消息
(5). 使用信道向队列发送消息
(6). 关闭信道，关闭连接

{% alert info %}
与simple队列相比，我们增加了队列和消息的持久化，确保rabbitmq server服务器停止时，队列和消息都不会丢失
{% endalert %}

消费者C代码:

```
 public function consumeWorkMQ()
    {
        //create a connection
        $connection = new AMQPStreamConnection('172.16.166.130','5672','guest','guest','/');
        //create a channel
        $channel = $connection->channel();
        //declare a queue for us to send to; then we can publish a message to the queue
        //set queue durable is ture，make sure that messages aren't lost
        $channel->queue_declare('hello',false,$durable=true,false,false);
                
        echo " [*] Waiting for messages. To exit press CTRL+C\n";

        $callback = function ($msg){
            echo '[x] Received ' . $msg->body . '\n';
            sleep(substr_count($msg->body, '.'));
            echo " [x] Done\n";
            //ack notification
            $msg->delivery_info['channel']->basic_ack($msg->delivery_info['delivery_tag']);
        };

        $channel->basic_qos(null, 1, null);
        //open ack 
        $channel->basic_consume('hello','',false,$no_ack=false,false,false,$callback);
        while($channel->is_consuming()){
            $channel->wait();
        }

        //close channel, close connection
        $channel->close();
        $connection->close();
    }

```

过程：
(1). 创建连接，设置服务地址，端口，账号，vhost
(2). 使用连接创建信道
(3). 使用信道创建队列
(4). 设置basic_qos值为1，告诉RabbitMQ在处理并确认前一个消息之前，不要向消费者发送新消息
(5). 创建消费者监听队列，从队列中读取消息,为了确保没有消息丢失，开启了ack机制（basic_consume方法第四个参数），在消息处理完成后回调basic_ack通知rabbitmq。
(6). 关闭信道，关闭连接

{% alert info %}
与simple队列相比，增加了消息处理完成ack确认，避免由于消费者意外中断导致消息未正确执行完成而丢失。多个消费者C时可能出现一些消费者特别忙，一些特别闲，但是这种情况RabbitMQ并不知道，还是均匀的发送消息（原因：This happens because RabbitMQ just dispatches a message when the message enters the queue. It doesn't look at the number of unacknowledged messages for a consumer. It just blindly dispatches every n-th message to the n-th consumer.），这样明显是不合理的，基于调度公平原则，设置basic_qos参数为1，直到Rabbitmq收到上次消息完成的确认再推送新消息给此消费者。
{% endalert %}

### 4.3 发布/订阅模式

![](https://raw.githubusercontent.com/heyuan110/static-source/master/media/rabbitmq/15649775451475.jpg)

功能：一个生产者P发送的消息会被多个消费者C消费。一个生产者，一个交换机，多个队列，多个消费者。

RabbitMQ种消息传递模型的核心思想是生产者永远不会将消息直接发送到队列，生产者甚至都不知道消息是否被发送到了队列。在消息传递模型中，生产者只能向交换机发送消息，交换机必须要准确知道消息将如果转发，这个转发规则类型有：direct,topic,headers,fanout，在上面的exchange type里已经详细解释。

4.1和4.2里我们没有提到交换机，为什么也可以给队列发消息？因为basic_publish在发送消息时其实用了默认的交换机(第二个参数设置为'')，设置为默认交换机时，消息会被路由到routing_key的队列（如果队列存在）。

如果消息发送到没有绑定队列的交换机，消息将会丢失，交换机本身不存储消息，消息存储在队列中.

生产者P代码:

```
    // https://www.rabbitmq.com/tutorials/tutorial-three-php.html
    public function publishMQ()
    {
        //create a connection
        $connection = new AMQPStreamConnection('172.16.166.130','5672','guest','guest','/');
        //create a channel
        $channel = $connection->channel();

        //create a exchange logs, type:fanout
        $channel->exchange_declare('logs', 'fanout', false, false, false);

        //create a message, publish it to the queue
        $data = $this->option('msg');
        if (empty($data)) {
            $data = "Hello World!";
        }
        $msg = new AMQPMessage(date("Y-m-d H:i:s ").$data);

        //send a message to exchange logs
        $channel->basic_publish($msg,'logs');

        echo " [x] Sent '".$data."'\n";

        //close channel, close connection
        $channel->close();
        $connection->close();
    }
```

过程：
(1). 创建连接，设置服务地址，端口，账号，vhost
(2). 使用连接创建信道
(3). 使用信道创建交换机，设置交换机类型为fanout（交换机接收的所有消息无差别转发到绑定队列）
(4). 创建消息,消息内容来自输入参数msg
(5). 使用信道向交换机发送消息，注意在这里我们没有设置routing_key，也没有创建任何队列
(6). 关闭信道，关闭连接

消费者C代码:

```
    public function subscribekMQ()
    {
        //create a connection
        $connection = new AMQPStreamConnection('172.16.166.130','5672','guest','guest','/');
        //create a channel
        $channel = $connection->channel();

        //create a exchange logs, type:fanout
        $channel->exchange_declare('logs', 'fanout', false, false, false);

        //declare a temporary queue for us to send to;  the temporary queue is auto delete when the consumer disconnect
        list($queue_name, ,) = $channel->queue_declare("", false, false, true, false);
                
        //bind queue to exchange logs
        $channel->queue_bind($queue_name, 'logs');

        echo " [*] Waiting for logs. To exit press CTRL+C\n";

        $callback = function ($msg) {
            echo ' [x] ', $msg->body, "\n";
        };

        //create consume
        $channel->basic_consume($queue_name, '', false, true, false, false, $callback);

        while($channel->is_consuming()){
            $channel->wait();
        }

        //close channel, close connection
        $channel->close();
        $connection->close();
    }
```

过程：
(1). 创建连接，设置服务地址，端口，账号，vhost
(2). 使用连接创建信道
(3). 使用信道创建交换机，设置交换机类型为fanout（交换机接收的所有消息无差别转发到绑定队列）
(4). 使用信道创建临时队列，队列名字设置为空，服务器会自动创建随机队列名称（大概类似：amq.gen-RAkn-INGuKex4JMNmoTZDA），一旦订阅消费者端口，临时队列会自动删除。返回队列名称$queue_name备用。
(5). 绑定队列交换机，让发送到交换机的消息都转发到队列
(6). 创建消费者监听队列，从队列中读取消息。
(7). 关闭信道，关闭连接

{% alert info %}
这个DEMO可以很好帮我们理解发布订阅模式，在Console里多启动几个订阅消费者，观察rabbitmq里队列会发现多了几个临时队列，如果终止它们随即会删除。
{% endalert %}

### 4.4 路由模式

![](https://raw.githubusercontent.com/heyuan110/static-source/master/media/rabbitmq/15649775511731.jpg)

功能：生产者P发送的消息到交换机并指定routing_key，消费者将队列绑定到交换机时需要指定路由key。

生产者P代码:

```
//https://www.rabbitmq.com/tutorials/tutorial-four-php.html
public function publishMQ()
    {
        //create a connection
        $connection = new AMQPStreamConnection('172.16.166.130','5672','guest','guest','/');
        //create a channel
        $channel = $connection->channel();

        //create a exchange logs, type:direct
        $exchange_name = 'direct_logs';
        $echange_type = 'direct';
        $channel->exchange_declare($exchange_name, $echange_type, false, false, false);

        //get routing key
        $routing_key = $this->option('level');
        if (empty($routing_key)) {
            $routing_key = "info";
        }

        //create a message, publish it to the queue
        $data = $this->option('msg');
        if (empty($data)) {
            $data = "Hello World!";
        }
        $msg = new AMQPMessage(date("Y-m-d H:i:s ").$data);

        //send a message to exchange logs
        $channel->basic_publish($msg,$exchange_name,$routing_key);

        echo ' [x] Sent ', $routing_key, ':', $data, "\n";

        //close channel, close connection
        $channel->close();
        $connection->close();
    }

```

过程：
(1). 创建连接，设置服务地址，端口，账号，vhost
(2). 使用连接创建信道
(3). 使用信道创建交换机，设置交换机类型为direct（根据routing_key转发消息）
(4). 创建消息,消息内容来自输入参数msg
(5). 使用信道向交换机发送消息，注意设置了routing_key
(6). 关闭信道，关闭连接

消费者C代码:
```
    public function subscribekMQ()
    {
        //create a connection
        $connection = new AMQPStreamConnection('172.16.166.130','5672','guest','guest','/');
        //create a channel
        $channel = $connection->channel();

        //create a exchange logs, type:fanout
        $exchange_name = 'direct_logs';
        $echange_type = 'direct';
        $channel->exchange_declare($exchange_name, $echange_type, false, false, false);

        //declare a temporary queue for us to send to;  the temporary queue is auto delete when the consumer disconnect
        list($queue_name, ,) = $channel->queue_declare("", false, false, true, false);

        $level = $this->option('level');
        $levels = [];
        if (empty($level)) {
            $levels = ['info'];
        }else{
            $levels = explode(',',$level);
        }
        //bind queue to exchange logs
        foreach ($levels as $routing_key) {
            $channel->queue_bind($queue_name,$exchange_name, $routing_key);
        }
                
        echo " [*] Waiting for logs. To exit press CTRL+C\n";

        $callback = function ($msg) {
            echo ' [x] ', $msg->delivery_info['routing_key'], ':', $msg->body, "\n";
        };

        //create consume
        $channel->basic_consume($queue_name, '', false, true, false, false, $callback);

        while($channel->is_consuming()){
            $channel->wait();
        }

        //close channel, close connection
        $channel->close();
        $connection->close();
    }
```

过程：
(1). 创建连接，设置服务地址，端口，账号，vhost
(2). 使用连接创建信道
(3). 使用信道创建交换机，设置交换机类型为direct（根据routing_key转发消息）
(4). 使用信道创建临时队列，队列名字设置为空，服务器会自动创建随机队列名称（大概类似：amq.gen-RAkn-INGuKex4JMNmoTZDA），一旦订阅消费者端口，临时队列会自动删除。返回队列名称$queue_name备用。
(5). 根据输入的routing_key绑定队列交换机，让发送到交换机的消息根据routing_key转发到队列
(6). 创建消费者监听队列，从队列中读取消息。
(7). 关闭信道，关闭连接

试着运行`php artisan consume:routing_mq --level="info,error,warnning"`和`php artisan consume:routing_mq --level="info"`，然后在生产者端输入不同的level看看，例如：
`php artisan produce:routing_mq --level=error --msg="error:test data"`和`php artisan produce:routing_mq --level=info --msg="info:test data"`.

### 4.5 通配符模式

![](https://raw.githubusercontent.com/heyuan110/static-source/master/media/rabbitmq/15649775560253.jpg)

功能：生产者P发送的消息到交换机并指定routing_key，并设置类型为topic，消费者将队列绑定到交换机时根据routing_key的值进行通配符匹配。

生产者P代码:

```
//https://www.rabbitmq.com/tutorials/tutorial-five-php.html
public function publishMQ()
    {
        //create a connection
        $connection = new AMQPStreamConnection('172.16.166.130','5672','guest','guest','/');
        //create a channel
        $channel = $connection->channel();

        //create a exchange logs, type:topic
        $exchange_name = 'topic_logs';
        $echange_type = 'topic';
        $channel->exchange_declare($exchange_name, $echange_type, false, false, false);

        //get routing key
        $routing_key = $this->option('routing_key');
        if (empty($routing_key)) {
            $routing_key = "anonymous.info";
        }

        //create a message, publish it to the queue
        $data = $this->option('msg');
        if (empty($data)) {
            $data = "Hello World!";
        }
        $msg = new AMQPMessage(date("Y-m-d H:i:s ").$data);

        //send a message to exchange logs
        $channel->basic_publish($msg,$exchange_name,$routing_key);

        echo ' [x] Sent ', $routing_key, ':', $data, "\n";

        //close channel, close connection
        $channel->close();
        $connection->close();
    }

```

过程：
(1). 创建连接，设置服务地址，端口，账号，vhost
(2). 使用连接创建信道
(3). 使用信道创建交换机，设置交换机类型为topic（根据routing_key转发消息）,注意和4.4里类型不一样。
(4). 创建消息,消息内容来自输入参数msg
(5). 使用信道向交换机发送消息，注意设置了routing_key
(6). 关闭信道，关闭连接

消费者C代码:

```
    public function subscribekMQ()
    {
        //create a connection
        $connection = new AMQPStreamConnection('172.16.166.130','5672','guest','guest','/');
        //create a channel
        $channel = $connection->channel();

        //create a exchange logs, type:topic
        $exchange_name = 'topic_logs';
        $echange_type = 'topic';
        $channel->exchange_declare($exchange_name, $echange_type, false, false, false);

        //declare a temporary queue for us to send to;  the temporary queue is auto delete when the consumer disconnect
        list($queue_name, ,) = $channel->queue_declare("", false, false, true, false);

        $binding_key = $this->option('binding_key');
        $binding_keys = [];
        if (empty($binding_key)) {
            $binding_keys = ['info'];
        }else{
            $binding_keys = explode(',',$binding_key);
        }
        //bind queue to exchange logs
        foreach ($binding_keys as $binding_key) {
            $channel->queue_bind($queue_name,$exchange_name, $binding_key);
        }
                
        echo " [*] Waiting for logs. To exit press CTRL+C\n";

        $callback = function ($msg) {
            echo ' [x] ', $msg->delivery_info['routing_key'], ':', $msg->body, "\n";
        };

        //create consume
        $channel->basic_consume($queue_name, '', false, true, false, false, $callback);

        while($channel->is_consuming()){
            $channel->wait();
        }

        //close channel, close connection
        $channel->close();
        $connection->close();
    }
```

过程：
(1). 创建连接，设置服务地址，端口，账号，vhost
(2). 使用连接创建信道
(3). 使用信道创建交换机，设置交换机类型为topic（根据routing_key转发消息）
(4). 使用信道创建临时队列，队列名字设置为空，服务器会自动创建随机队列名称（大概类似：amq.gen-RAkn-INGuKex4JMNmoTZDA），一旦订阅消费者端口，临时队列会自动删除。返回队列名称$queue_name备用。
(5). 根据输入的routing_key规则绑定队列交换机，让发送到交换机的消息根据routing_key通配符规则转发到队列
(6). 创建消费者监听队列，从队列中读取消息。
(7). 关闭信道，关闭连接

{% alert info %}
 通配符模式下生产者和消费者的routing_key通配符必须是.号分隔的单词列表，例如生产者的routing_key设置为lazy.white.rabbit，与消费者routing_key设置为lazy.*.rabbit的队列匹配。
 
替换规则（注意代替的是单词，不是字符）：
- *(星号)代替一个单词，
- #(井号)代替零个或多个单词

当topic的routing_key设置为一个"#"时，会接收所有消息，类似fanout交换机类型。
当topic的routing_key不包含通配符*或#时，类似direct交换机类型。
{% endalert %}

先运行消费者：

```
php artisan consume:topic_mq --binding_key="*.white.dog"
php artisan consume:topic_mq --binding_key="test-key"
php artisan consume:topic_mq --binding_key="lazy.*.rabbit"
php artisan consume:topic_mq --binding_key="lazy.#"
php artisan consume:topic_mq --binding_key="*"
php artisan consume:topic_mq --binding_key="#"
```

随后运行生产者观察消费者端输出

```
php artisan produce:topic_mq --routing_key="quick.red.fox" --msg="test1"
php artisan produce:topic_mq --routing_key="lazy.red.fox" --msg="test2"
php artisan produce:topic_mq --routing_key="lazy.red.fox.test" --msg="test3"
php artisan produce:topic_mq --routing_key="quick.white.dog" --msg="test4"
php artisan produce:topic_mq --routing_key="test-key" --msg="test5"
php artisan produce:topic_mq --routing_key="test" --msg="test6"
```

通过上面生产和消费消息演示，可以快速加深理解topic通配符模式。

### 4.6 RPC模式

![](https://raw.githubusercontent.com/heyuan110/static-source/master/media/rabbitmq/15649775606069.jpg)

RPC工作过程：

- 当客户端启动时，创建一个匿名的独占回调队列
- 对于RPC请求，客户端发送带有两个属性的消息,reply_to设置为回调队列，correlation_id设置为每个请求的唯一值。
- 请求发送到rpc_queue队列
- RPC服务端获取到rpc_queue队列上的请求后，执行完成，将结果通过reply_to设置的队列返回给客户端
- 客户端收到消息后，检查消息的correlation_id属性，如果和请求中的值匹配，则将响应返回给应用程序。

{% alert info %}
在AMQP0-9-1协议中消息带有14个属性，大多数很少使用，但下面几个需要了解：

1. `delivery_mode`：设置为2表示消息持久化，设置为1表示临时消息
2. `content_type`: 消息类型，例如json格式的可设置为：`application / json`
3. `reply_to`: 通常用于命名回调队列
4. `correlation_id`: 将RPC的请求和响应关联起来

本节参考：<https://www.rabbitmq.com/tutorials/tutorial-six-php.html>

{% endalert %}

[Client](https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/php/rpc_client.php)和[Server](https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/php/rpc_server.php)代码查看。

## 5. RabbitMQ安装和集群配置

### 5.1 Docker安装

适合本地学习，安装简单快速，执行：

`docker run -d -p 5672:5672 -p 15672:15672 --name rabbitmq rabbitmq:management`

rabbitmq:management的镜像200M左右，安装完成可以在浏览器访问http://ip:15672/,默认账号密码guest/guest

### 5.2 本地安装

Ubuntu16.04安装Shell脚本install.sh

直接执行

```
sh -c "$(curl -fsSL https://raw.githubusercontent.com/heyuan110/BashShell/master/rabbitmq_install.sh)"
```

或者拷贝源码自己建立sh文件

```
#!/bin/sh

# Install RabbitMQ signing key
curl -fsSL https://github.com/rabbitmq/signing-keys/releases/download/2.0/rabbitmq-release-signing-key.asc | sudo apt-key add -

# Install apt HTTPS transport
sudo apt-get install apt-transport-https

# Add Bintray repositories that provision latest RabbitMQ and Erlang 21.x releases
sudo tee /etc/apt/sources.list.d/bintray.rabbitmq.list <<EOF
deb http://dl.bintray.com/rabbitmq-erlang/debian xenial erlang
deb https://dl.bintray.com/rabbitmq/debian xenial main
EOF

# Update package indices
sudo apt-get update -y

# Install rabbitmq-server and its dependencies
sudo apt-get install rabbitmq-server -y --fix-missing

# Start RabbitMQ Server
# sudo service rabbitmq-server start

# RabbitMQ Server Status
# sudo service rabbitmq-server status

# Stop RabbitMQ Server
# sudo service rabbitmq-server stop

# Restart RabbitMQ Server
# sudo service rabbitmq-server restart

```

执行`sh install.sh`等待安装完成。刚搭建好的rabbitmq server没有任何配置文件，增加配置文件`vi /etc/rabbitmq/rabbitmq.config`,输入下面内容:

```
[{rabbit, [{loopback_users, []}]}].
```
rabbitmq默认搭建好有一个用户名guest，密码guest的用户，但只能local访问。上面的配置开启guest可外部访问,配置完重启rabbitmq。

Rabbit主要脚本在`/usr/sbin/`目录下，执行`ll rabbit*`查看:

```
root@rmq2:/usr/sbin# ll rabbitmq*
-rwxr-xr-x 1 root root 3508 Jul 29 10:37 rabbitmq-diagnostics*
-rwxr-xr-x 1 root root 3508 Jul 29 10:37 rabbitmq-plugins*
-rwxr-xr-x 1 root root 3508 Jul 29 10:37 rabbitmq-server*
-rwxr-xr-x 1 root root 3508 Jul 29 10:37 rabbitmqctl*
```

分为交互，服务，插件几个命令。

**WEB管理插件**

为了方便管理，RabbitMQ官方提供了一套WEB界面，需要单独开启。

开启：`rabbitmq-plugins enable rabbitmq_management`
禁用：`rabbitmq-plugins disable rabbitmq_management`

打开浏览器输入<http://192.168.8.131:15672>，输入guest/guest账号密码.
![](https://raw.githubusercontent.com/heyuan110/static-source/master/media/rabbitmq/15664687172489.jpg)

### 5.3 用户管理

可以在WEB管理界面直接操作，也可以用下面的命令行操作。

#### 5.3.1 添加用户

格式：`rabbitmqctl add_user <username> <newpassword>`
例子：创建一个bruce用户，密码为123456，这时用户没有角色和权限。
```
rabbitmqctl add_user bruce 123456
```

#### 5.3.2 查看用户列表

格式：`rabbitmqctl list_users`
例子：
```
root@rmq1:/usr/sbin# rabbitmqctl list_users
Listing users ...
user	tags
bruce1	[administrator]
guest	[administrator]
bruce2	[administrator]
```

#### 5.3.3 修改密码

格式：`rabbitmqctl change_password <username> <newpassword>`
例子：修改bruce用户密码`rabbitmqctl  change_password  bruce 0123456`

#### 5.3.4 删除用户

格式：`rabbitmqctl delete_user <username>`
例子：删除bruce用户`rabbitmqctl  delete_user bruce`

#### 5.3.4 添加角色

格式：`rabbitmqctl set_user_tags <username> <tag>`
例子：设置bruce为管理员`rabbitmqctl set_user_tags bruce administrator`

上面新创建的bruce用户没有用户角色，无法访问web插件等，按上面例子设置为管理员。rabbitmq用户角色分为五类：

- 超级管理员(administrator)
可登陆管理控制台(启用management plugin的情况下)，可查看所有的信息，并且可以对用户，策略(policy)进行操作。

- 监控者(monitoring)
可登陆管理控制台(启用management plugin的情况下)，同时可以查看rabbitmq节点的相关信息(进程数，内存使用情况，磁盘使用情况等)

- 策略制定者(policymaker)
可登陆管理控制台(启用management plugin的情况下), 同时可以对policy进行管理。但无法查看节点的相关信息。

- 普通管理者(management)
仅可登陆管理控制台(启用management plugin的情况下)，无法看到节点信息，也无法对策略进行管理。

- 其他
无法登陆管理控制台，通常就是普通的生产者和消费者。

#### 5.3.5 删除角色

格式：`rabbitmqctl clear_permissions [-p VHostPath] <username>`
例子：清除bruce的角色`rabbitmqctl  clear_permissions  -p /  bruce`

#### 5.3.4 用户授权

格式：`rabbitmqctl set_permissions [-pvhostpath] {user} {conf} {write} {read}`
例子：上面bruce用户设置角色后没有赋予权限，这时bruce用户只能本地登录。赋予bruce用户对vhost/中所有资源的配置，写，读权限，方便管理其中资源。
```
rabbitmqctl set_permissions -p "/"bruce ".*" ".*" ".*"
```

#### 5.3.5 查看用户授权

格式：`rabbitmqctl list_permissions [-p VHostPath]`
例子：查看vhost / 下所有用户的权限

```
rabbitmqctl list_permissions -p /
```

### 5.4 集群配置

RabbitMQ有三种模式，其中两种集群模式:

- 单机模式：不做集群，单机运行。

- 普通模式：默认集群模式，集群的所有节点具有相同元数据(队列结构)，但消息实体只存在于集群中一个节点，当存消息实体的节点崩溃后消息不可用。举例rmq1，rmq2，rmq3三个节点，生产者先在rmq2上declare一个队列test_queue2，生产者连rmq1发布消息到集群的test_queue2队列，假设消费者连rmq3消费test_queue2，rmq1突然down掉，不影响消息消费，但如果rmq2 down掉，这时消费消息报错。这种架构的缺点很明显非高可用，如果保存实体消息的机器down掉不能恢复或未做持久化，消息就丢失。上面的例子实体消息存在rmq2，消费者连接的rmq3，消息怎么传递的呢？当消费者连接rmq3时，rabbitmq会临时在rmq2、rmq3间进行消息传输，rmq3再传递给消费者。

![](https://raw.githubusercontent.com/heyuan110/static-source/master/media/rabbitmq/15669015983876.jpg)

- 镜像模式：把需要的队列做成镜像队列，元数据（queue结构）和实体数据（message）存在于多个节点，属于RabbitMQ的[HA(Highly Available)](https://www.rabbitmq.com/ha.html)方案.与普通模式相比，消息实体会主动在镜像节点之间同步，而不是消费者拉取数据时临时传输。镜像队列是基于普通的集群模式的,所以还是得先配置普通集群,然后才能设置镜像队列。镜像队列设置后，会分一个主节点和多个从节点，如果主节点宕机，从节点会有一个选为主节点，原先的主节点起来后会变为从节点。queue和message虽然会存在所有镜像队列中，但客户端读取时不论物理面连接的主节点还是从节点，都是从主节点读取数据，然后主节点再将queue和message的状态同步给从节点，因此多个客户端连接不同的镜像队列不会产生同一message被多次接受的情况。


#### 5.3.1 配置集群普通模式

- (1)准备环境
三台机器：rmq1,rmq2,rmq3，对应IP分别为：192.168.8.131、192.168.8.146、192.168.9.123
环境：Ubuntu16.04

- (2)安装Rabbitmq
用上面[5.2本地安装](#5-2_本地安装)的方法在每台机器安装好RabbitMQ，并确认可以运行。开启每一台的web管理插件，并只在rmq1上创建新用户bruce/123456，授予管理员角色和权限，后面rmq2，rmq3加入集群后直接同步rmq1的用户数据。

- （3)统一集群节点cookie
Rabbitmq集群依赖erlang集群，而erlang集群式通过cookie进行通信认证，因此配置Rabbitmq集群第一步统一节点之间的cookie，默认cookie保存在文件`/var/lib/rabbitmq/.erlang.cookie`，400只读权限。
在rmq1上执行`cat /var/lib/rabbitmq/.erlang.cookie`查看cookie值，复制出来待用。
在rmq2上调整`.erlang.cookie`文件权限为可写入600,`sudo chmod 600 /var/lib/rabbitmq/.erlang.cookie`，然后编辑cookie值`vi /var/lib/rabbitmq/.erlang.cookie`，将rmq1上复制出来的cookie值粘贴进去，`:wq`保存退出。将cookie文件权限调整为只读400，`sudo chmod 400 /var/lib/rabbitmq/.erlang.cookie`,在rmq3上同样的操作。
然后重启rmq2,rmq3的rabbitmq，执行`sudo service rabbitmq-server restart`,
这时可以分别在3台机器查看rabbitmq集群状态,`rabbitmqctl cluster_status`

- （4)将三个节点组成集群
rabbitmq-server在启动时，会一起启动节点和应用，预先设置了rabbitmq应用为standalone模式。要将一个节点加入到现有的集群中，需要停止这个应用并将节点设置为原始状态，然后再加入集群。
组建集群的方式：将rmq2加入到rmq1,将rmq3加入到rmq1
{% alert warning %}
注意：rabbitmq集群要求节点之间可以连通，`rabbitmqctl join_cluster rabbit@rmq1`中`rabbit@rmq1`的rmq1指域名（我这里直接用的hostname)，这里它解析到当前节点本地IP.打开`vi /etc/hosts`查看域名与IP对应关系，例如我们这次创建的3个节点，应该有这样的对应关系在'/etc/hosts'文件里：
192.168.9.123       rmq3
192.168.8.146       rmq2
192.168.8.131       rmq1
{% endalert %}
首先再rmq2上执行`rabbitmqctl stop_app`停止应用
将rmq2加入到rmq1执行`/usr/sbin/rabbitmqctl join_cluster rabbit@rmq1`
启动rmq2节点应用执行`rabbitmqctl start_app`
这时rmq1和rmq2就组成一个集群了，在任意节点执行`rabbitmqctl cluster_status`可以查看集群状态：
```
root@rmq2:/home/bruce# rabbitmqctl cluster_status
Cluster status of node rabbit@rmq2 ...
[{nodes,[{disc,[rabbit@rmq1,rabbit@rmq2]}]},
 {running_nodes,[rabbit@rmq1,rabbit@rmq2]},
 {cluster_name,<<"test-rabbitmq-cluster">>},
 {partitions,[]},
 {alarms,[{rabbit@rmq1,[]},{rabbit@rmq2,[]}]}]
```

rmq1和rmq2在一个集群，rmq1上的用户信息也同步给rmq2了，我们可以在web管理界面上刷新看一看
![](https://raw.githubusercontent.com/heyuan110/static-source/master/media/rabbitmq/15670857248129.jpg)
依照上面的步骤添加rmq3到rmq1.至此我们可以写demo验证一下集群普通模式下的元数据和实体消息传递方式，试着让不同节点down掉看看。

#### 5.3.2 配置集群镜像模式

前面对于镜像模式做了说明，我们配置好了集群普通模式情况下，给集群启用策略。只需要在集群中任意节点启用策略，策略会自动同步到集群其它节点：

设置策略命令格式：

```
set_policy [-p vhostpath] {name} {pattern} {definition} [priority]

解释：
-p Vhost： 可选参数，针对指定vhost下的queue进行设置
Name: policy的名称
Pattern: queue的匹配模式(正则表达式)
Definition：镜像定义，包括三个部分ha-mode, ha-params, ha-sync-mode
        ha-mode:指明镜像队列的模式，有效值为 all/exactly/nodes
        all：表示在集群中所有的节点上进行镜像
        exactly：表示在指定个数的节点上进行镜像，节点的个数由ha-params指定
        nodes：表示在指定的节点上进行镜像，节点名称通过ha-params指定
        ha-params：ha-mode模式需要用到的参数
        ha-sync-mode：进行队列中消息的同步方式，有效值为automatic和manual，默认manual
priority：可选参数，policy的优先级
```

在任意节点执行：`rabbitmqctl set_policy ha-all "^" '{"ha-mode":"all","ha-sync-mode":"automatic"}'`，“^” 表示所匹配所有队列名称,"^test"为匹配名称为test的exchanges或者queue。

镜像模式下集群重启时，最后一个挂掉的节点应该第一个重启，如果因特殊原因（比如同时断电），而不知道哪个节点最后一个挂掉。可用以下方法重启：

先在一个节点上执行
```
$ rabbitmqctl force_boot
$ sudo service rabbitmq-server start
```

在其他节点上执行
```
$ sudo service rabbitmq-server start
```

查看cluster状态是否正常（要在所有节点上查询）。
```
rabbitmqctl cluster_status
```

{% alert info %}
我查了官方资料没有找到怎么看谁是master节点谁是slave节点，有一个小技巧可以试试，
使用`rabbitmqctl list_queues name pid slave_pids synchronised_slave_pids`命令
，其中name是队列名字，pid为master节点，slave_pids返回的数组里都是从节点，synchronised_slave_pids是slave节点同步情况。举个例子，集群(HA)现在有三个队列：hello、hello2、hello3，hello和hello3里没有消息，hello2队列里有6条记录，按照上面的步骤给集群增加一个rmq4节点，镜像队列中默认消息不会主动同步到新节点，这时用上面命令查看可以看到结果:

![](https://raw.githubusercontent.com/heyuan110/static-source/master/media/rabbitmq/15671540122491.jpg)

很明显hello2消息没有同步到rmq4.
{% endalert %}

#### 5.3.3 配置HAProxy

基于上面例子加入HAProxy后画一个整体拓扑图如下：
![](https://raw.githubusercontent.com/heyuan110/static-source/master/media/rabbitmq/15674979648150.jpg)

采用VMware ESXi虚拟出的4台服务器，HAProxy没开新机器，和rabbit@rmq1在一台上。四个节点中2个RAM，两个DISC（如果是3个节点建议2个DISC，1个RAM）。

HAProxy安装：`sudo apt-get install haproxy`,安装完成查看版本`haproxy -v`,打开haproxy配置文件`vi /etc/haproxy/haproxy.cfg`,

```
global
        log /dev/log    local0
        log /dev/log    local1 notice
        chroot /var/lib/haproxy
        stats socket /run/haproxy/admin.sock mode 660 level admin
        stats timeout 30s
        user haproxy
        group haproxy
        daemon

        # Default SSL material locations
        ca-base /etc/ssl/certs
        crt-base /etc/ssl/private

        # Default ciphers to use on SSL-enabled listening sockets.
        # For more information, see ciphers(1SSL). This list is from:
        #  https://hynek.me/articles/hardening-your-web-servers-ssl-ciphers/
        ssl-default-bind-ciphers ECDH+AESGCM:DH+AESGCM:ECDH+AES256:DH+AES256:ECDH+AES128:DH+AES:ECDH+3DES:DH+3DES:RSA+AESGCM:RSA+AES:RSA+3DES:!aNULL:!MD5:!DSS
        ssl-default-bind-options no-sslv3
        
defaults
        log     global
        mode    http
        option  httplog
        option  dontlognull
        timeout connect 5000
        timeout client  50000
        timeout server  50000
        errorfile 400 /etc/haproxy/errors/400.http
        errorfile 403 /etc/haproxy/errors/403.http
        errorfile 408 /etc/haproxy/errors/408.http
        errorfile 500 /etc/haproxy/errors/500.http
        errorfile 502 /etc/haproxy/errors/502.http
        errorfile 503 /etc/haproxy/errors/503.http
        errorfile 504 /etc/haproxy/errors/504.http

####################################################################
listen http_front
        bind 0.0.0.0:1080
        mode http
        stats refresh 30s
        stats uri /haproxy?stats
        stats realm Haproxy Manager
        stats auth admin:admin
        #stats hide-version

####################################################################
listen rabbitmq_admin
    bind 0.0.0.0:8004
    server node1 192.168.8.131:15672
    server node2 192.168.8.146:15672
    server node3 192.168.9.123:15672
    server node3 192.168.9.226:15672

####################################################################
listen rabbitmq_cluster
    bind 0.0.0.0:5678
    option tcplog
    mode tcp
    timeout client  3h
    timeout server  3h
    option          clitcpka
    balance roundrobin
    #balance url_param userid
    #balance url_param session_id check_post 64
    #balance hdr(User-Agent)
    #balance hdr(host)
    #balance hdr(Host) use_domain_only
    #balance rdp-cookie
    #balance leastconn
    #balance source //ip
    server   node1 192.168.8.131:5672 check inter 5s rise 2 fall 3
    server   node2 192.168.8.146:5672 check inter 5s rise 2 fall 3
    server   node3 192.168.9.123:5672 check inter 5s rise 2 fall 3
    server   node3 192.168.9.226:5672 check inter 5s rise 2 fall 3

```

保存重启HAProxy:`sudo service haproxy restart`。
打开HAProxy管理页面（账号密码admin/admin）：http://192.168.8.131:1080/haproxy?stats
打开RabbitMQ集群地址:http://192.168.8.131:8004
RabbitMQ集群服务地址:192.168.8.131:5678，可以写demo测试下。

#### 5.3.3.1 配置AWS ELB

如果生产环境使用AWS EC2搭建RabbitMQ集群，可以利用ELB做负载均衡,在TCP层做转发，选择网络负载均衡类型.

![](https://raw.githubusercontent.com/heyuan110/static-source/master/media/rabbitmq/rabbitmq-elb-flow.png)

rabbitmq在生产环境部署，为了安全选择internal类型，不提供对外访问。选择TCP protocol，端口5672。注意选择可用区和子网，rabbitmq机器和elb可用区子网相同。

Target Group的health checks选择tcp，traffic port。

对搭建了rabbitmq的EC2赋予相关端口访问权限。
![](https://raw.githubusercontent.com/heyuan110/static-source/master/media/rabbitmq/E05FF76B-4A4C-4626-BFF1-E895E9FAA10D.png)

端口相关功能介绍参考<https://www.rabbitmq.com/networking.html>

#### 5.3.4 集群镜像队列中节点down机

当slave节点down掉，除了与slave已经连接的客户端中断，不会有其它影响。
当master节点down掉，会出现下面连锁反应：
(1)与master相连的客户端中断；
(2)选举最老的slave节点为master，因为最老的slave与前任master之间的同步状态应该是最好的。若此时所有slave处于未同步状态，则未同步消息会丢失
(3)新master节点requeue所有unack消息，因为新master节点无法区分这些unack消息是否已经到达客户端，或是ack消息丢失在老master链路上，或是丢失在master组播ack消息到所有slave节点的链路上。所以基于消息可靠性，会在新master上requeue所有unack的消息，造成的后果就是客户端可能会有重复消息（客户端处理消息最好考虑幂等性）。
(4)设置了`x-cancel-on-ha-failover`参数为true的客户端会收到Consumer Cancellation Notification通知。从镜像队列消费消息的客户端如果想要知道它们消费消息的队列是否failover（故障转移）,可以将`x-cancel-on-ha-failover`设置为true，当镜像队列发生failover后，关于message已被发送到哪个客户端的信息将丢失，所有未ack的message都会被requeue(也叫redelivered flag)后重新发送。如果不设置那么客户端就无法感知到master宕机，会一直等待。

从上面可以看出集群中存在镜像队列时，重选master节点有风险。

#### 5.3.5 集群镜像队列中节点启动顺序

镜像队列中最后一个停止的节点会是master，启动顺序必须是master先启动，如果slave先启动，它会有30s的等待时间，等待master的启动，然后加入cluster中（如果30s内master没有启动，slave会自动停止）。当所有节点因故（断电等）同时离线时，每个节点都认为自己不是最后一个停止的节点。要恢复镜像队列，可以尝试在30s之内启动所有节点。

## 6. RabbitMQ问题集

### 6.1 如何选择RabbitMQ的节点类型？

RabbitMQ对queue中message的保存有两种方式：disc和ram。
如果采用disc，则需要对exchange／queue／delivery mode都要设置成durable模式。
使用disc方式的好处是当RabbitMQ失效了，message仍然可以在重启之后恢复；
使用ram方式，RabbitMQ处理message的效率要高很多，ram和disc两种方式的效率比大概是3:1。
所以如果在有其它HA手段保障的情况下，选用ram方式是可以提高消息队列的工作效率的。

将节点从disc类别切换成ram（反之相同）
```
rabbitmqctl stop_app
rabbitmqctl change_cluster_node_type ram
rabbitmqctl start_app
```
### 6.2 消息发送的速率超过了RabbitMQ的处理能力怎么办？
RabbitMQ会自动减慢这个连接的速率，让client端以为网络带宽变小了，发送消息的速率会受限，从而达到流控的目的。使用`rabbitmqctl list_connections`查看连接，如果状态为“flow”，则说明这个连接处于flow-control状态。

*相关参考:*
*[1.RabbitMQ Admin Guide](https://www.rabbitmq.com/admin-guide.html)*
*[2.RabbitMQ手册之rabbitmqctl](https://www.jianshu.com/p/61a90fba1d2a?utm_campaign)*
*[3.Ubuntu16.04搭建Rabbitmq集群](https://www.jianshu.com/p/498c63e4ace1)*
*[4.RabbitMQ+HAProxy构建高可用消息队列](https://blog.csdn.net/qq_34039315/article/details/77619736)*
*<https://blog.csdn.net/u013256816/article/details/71097186>*

