# 前言

本文讲述的只是主要是 `RabbitMQ` 的入门知识，学习本文主要可以掌握以下知识点：

- MQ 的发展史
- AMQP 协议
- Rabbit MQ 的安装
- Rabbit MQ 在 Java API 中的使用
- RabbitMQ 与 SpringBoot 的集成

# MQ 的诞生历史

大部分技术的刚产生时适用范围都是特定的。比如互联网的产生，刚开始出现的通信协议各个产商之间是无法兼容的，随着历史的发展，产生了业内的通信标准`tcp/ip`协议，而`MQ`也是一样，第一款 `MQ` 类软件是由一个在美国印度人 `Vivek Ranadive` 创办的一家公司 `Teknekron`，并实现了世界上第一个消息中间件 `The Information Bus(TIB)`。

随着第一款`MQ` 类软件`TIB`的诞生，各大厂商立刻跟进，百花争鸣，涌现了一批`MQ`类软件，比如`IBM`开发的`IBM Wesphere`，微软开发的`MSMQ`等等，但是正因为标准不统一，这就给我们使用者带来了很大的不便，每次切换`MQ`时都需要重复去实现不同的协议和不同API的调用。

`2001`年，`Java`语言的老东家`Sun`公司发布了一个`JMS`规范，其在各大产商的`MQ`上进行了统一封装，使用者如果需要使用不同的`MQ`只需要选择不同的驱动就可以了（和我们使用数据库驱动一个道理）。`JMS`规范虽然统一了标准，但是`JMS`规范却有一个很大的缺陷就是它是和`Java`语言进行绑定的，所以依然没有从根本上解决问题。

`2004`年，`AMQP`规范出现了，真正做到了跨语言和跨平台，自此`MQ`迎来了发展的黄金时代。

`2007`年，`Rabbit`公司基于`AMQP`规范开发出了一款消息队列`RabbmitMQ`。很快的，`RabbitMQ`就得到了大家的喜爱，被用户广泛使用。

# 什么是 MQ

`MQ`即：`Message Queue`，称之为消息队列或者消息中间件。`MQ`的本质是：使用高效可靠的消息传递机制来进行与平台无关的数据传递，并基于数据通信来进行分布式系统的集成。也就是说`MQ`主要是用来解决消息的通信问题，其主要有以下三个特点：

- 1、`MQ`是一个独立运行的服务。通过生产者来发送消息，使用消费者来接收消费。
- 2、内部采用了队列来进行消息存储，一般采用的均是先进先出（`FIFO`）队列。
- 3、具有发布订阅的模型，消费者可以根据需要来获取自己想要的消息。  

# 为什么需要 MQ

以`Java`语言为例，`JDK`本身就提供了许多不同类型的队列，那么为什么还需要用`MQ`呢？这是因为：

- 1、跨语言。各大编程语言内部实现的队列是和语言绑定的，而且是单机的，在分布式环境下无法很好的工作，所以我们需要可以单独部署不依赖于语言的`MQ`。
- 2、异步解耦。消息队列可以实现异步通信，这样发送消息方只需要关心消息是否发送成功，而接受消息方只需要关心怎么处理队列中的消息，实现了消费和生产者的解耦。
- 3、流量削峰。因为消息队列是先进先出，所以如果把需要消费的消息放进队列，那么消费者就可以避免被瞬间大流量击垮，而是可以从容的根据自己的能力从队列中取出消息进行消费。

# RabbitMQ

`RabbitMQ` 中的 `Rabbit` 是兔子的意思，就是形容跑的和兔子一样快。其是一款轻量级的，支持多种消息传递协议的高可用的消息队列。`RabbitMQ` 是由 `Erlang` 语言编写的，而 `Erlang` 语言就是一款天生适合高并发的语言。

## RabbitMQ 的优势和特性

`RabbitMQ` 作为一款非常流行的消息中间件，其有着非常丰富的特性和优势：

- 高可靠性：`RabbitMQ` 提供了持久化、发送应答、发布确认等功能来保证其可靠性。  
- 灵活的路由：通过不同的交换机（Exchange）来实现了消息的灵活路由。
- 集群与扩展性：多个节点可以组成一个逻辑上的服务器，支持负载。
- 高可用性：通过镜像队列实现了队列中数据的复制，保证了在极端情况下部分节点出现 `crash` 整个集群仍然可用。
- 支持多种协议：`RabbitMQ` 最初是为了支持 `AMQP` 协议而开发的，所以 `AMQP` 是其核心协议，但是其也支持其他如：`STOMP`，`MOTT`，`HTTP` 等协议。
- 支持多客户端：`RabbitMQ` 几乎支持所有常用语言客户端，如：`Java`，`Python`，`Ruby`，`Go` 等。
- 丰富的插件系统：支持各种丰富的插件扩展，同时也支持自定义插件。比如最常用的 `RabbitMQ` 后台管理系统就是以插件的形式实现的。

## AMQP 模型

[AMQP](https://www.amqp.org/) 全称是：Advanced Message Queuing Protocol。`RabitMQ` 最核心的协议就是基于 `AMQP` 模型的 `AMQP` 协议，`AMQP` 模型目前最新的版本是 `1.0` 版本，但是目前官方推荐使用者的最佳版本仍是基于 `0.9.1` 版本的 `AMQP` 模型，`0.9.1` 版本在 `RabbitMQ`官网中也将其称之为 `AMQP 0-9-1`模型。

`AMQP 0-9-1`（高级消息队列协议）是一种消息传递协议，它允许符合标准的客户端应用程序与符合标准的消息传递中间件代理进行通信。消息传递代理（Broker）从发布者（Publisher，即发布消息的应用程序，也称为生产者：Producter）接收消息，并将其路由到使用者（消费者：Consumer，即处理消息的应用程序）。

`AMQP 0-9-1` 模型的核心思想为：消息被发布到交换处，通常被比作邮局或邮箱。然后，交换机使用称为绑定的规则将消息副本分发到队列。然后，代理将消息传递给订阅了队列的使用者，或者使用者根据需要从队列中获取/提取消息。

下图就是一个 `AMQP` 模型简图，理解了这幅图，那么就基本理解了 `RabbitMQ` 的工作模式。

![AMQP-0-9-1模型](image/1/AMQP-0-9-1.png)

### Producer 和 Consumer 

`Producer ` 即生产者，一般指的是应用程序客户端，生产者会产生消息发送给 `RabbitMQ`，等待消费者进行处理。

`Consumer` 即消费者，消费者会从特定的队列中取出消息，进行消费。当消息传递给消费者时，消费者会自动通知 `Broker`，`Broker` 只有在收到关于该消息的通知时才会从队列中完全删除该消息。

### Connection：我是一个 TCP 长连接

生产者发送消息和消费者接收消息之前都必须要和 `Broker` 建立一个 `tcp` 长连接，才能进行通信。

### Channel：我是被虚构出来的

消息队列的作用之一就是用来做削峰，所以消息队列在高并发场景可能会有大量的生产者和消费者，那么假如每一个生产者在发送消息时或者每一个消费者在消费消息时都需要不断的创建和销毁 `tcp` 连接，那么这对 `Broker` 会是一个很大的消耗，为了降低这个 `tcp` 连接的创建频率，`AMQP` 模型引入了 `Channel`（通道或者信道）。

`Channel` 是一个虚拟的的连接，可以被认为是“轻量级的连接，其共享同一个 `tcp` 连接”。在同一个 `tcp` 长连接里面可以通过创建和销毁不同的 `Channel` 来减少了创建和销毁 `tcp` 连接的频率，从而大大减少了资源的消耗。

客户端（生产者/消费者）执行的每个协议操作都发生在通道上。特定 `Channel` 上的通信完全独立于另一个 `Channel` 上的通信，因此每个协议方法还携带一个`Channel ID`（又称通道号）。

`Channel` 只存在于连接的上下文中，不会独立存在，所以当一个 `tcp` 连接被关闭时，其中所有 `Channel` 也都会被关闭。

`Channel` 是线程不安全的，所以对于使用多个线程/进程进行处理的应用程序，需要为每个线程/进程创建一个 `Channel`，而不是共享同一个 `Channel`。

### Broker：我只是一个普通的代理商

`Broker` 直接翻译成中文就是：中介/代理，所以如果我们使用的是 `RabbitMQ`，那么这个 `Broker` 就是指的 `RabbitMQ` 服务端。

### Exchange：我只是做一个消息的映射

`Echange` 即交换机，因为要实现生产者和消费者的多对多关系，所以只有一个队列是无法满足要求的，那么如果有多个队列，每次我们发送的消息应该存储到哪里呢？交换机就是起到了中间角色的作用，我们发送消息到交换机上，然后通过交换机发送到对应的队列，交换机和队列之间需要提前绑定好对应关系，这样消息就到了各自指定的队列内，然后消费者就可以直接从各自负责的队列内取出消息进行消费。

### Queue：我才是真正存储消息的地方

消息发送到 `Broker` 之后，通过交换机的映射，存储到指定的 `Queue`里面。

### VHost：我只是一个命名空间而已

`VHost` 类似于命名空间，主要作用就是用来隔离数据的，比如我们由很多个业务系统都需要用到 `RabbitMQ`，如果是一台服务器完全可以满足要求，那就没必要安装多个 `RabbitMQ` 了，这时候就可以定义不同的 `VHost` ，不同的 `VHost` 就可以实现各个业务系统间的数据隔离。

## RabbitMQ 的安装

`RabbitMQ` 是用 `Erlang` 语言开发的，所以在安装 `RabbitMQ` 之前，需要先安装 `Erlang`，`RabbitMQ` 和 `Erlang` 之间有[版本对应关系](https://www.rabbitmq.com/which-erlang.html)，这个需要注意，本文以 `Erlang 21.3` 和 `RabbitMQ3.8.4` 为例进行安装 。

1. 安装 `Erlang` ：

```java
yum -y install gcc glibc-devel make ncurses-devel openssl-devel xmlto perl wget //提前安装一些依赖，个人电脑依赖不同，可根据实际情况选择未安装的依赖进行安装
wget http://erlang.org/download/otp_src_21.3.tar.gz # 下载（也可以下载好传到服务器）
tar -xvf otp_src_21.3.tar.gz //解压
mkdir erlang //在指定目录，如/usr/local下创建erlang目录
cd otp_src_21.3 //切换到解压后的目录
./configure --prefix=/usr/local/erlang //编译（路径根据实际情况选择）
make && make install //安装
```

2. 配置 `Erlang` 环境变量：

```java
vim /etc/profile  //编辑环境变量文件（CentOS系统默认环境变量文件，其他系统可能不一样）
export PATH=$PATH:/usr/local/erlang/bin //在末尾加入环境变量配置（路径根据实际情况选择）
source /etc/profile //实时生效
```

3. 输入 `erl` 验证 `Erlang` 是否安装成功。如果出现如下显示版本号的界面则说明安装成功（可以输入 `halt().` 命令进行退出）：

![验证erl](image/1/验证erlang安装.png)

4. 安装 `RabbitMQ`：

```java
wget https://dl.bintray.com/rabbitmq/all/rabbitmq-server/3.8.4/rabbitmq-server-generic-unix-3.8.4.tar.xz  //下载RabbitMQ
xz -d rabbitmq-server-generic-unix-3.8.4.tar.xz //解压
tar -xvf rabbitmq-server-generic-unix-3.8.4.tar //解压
```

5. 同样的，这里需要进行环境变量配置：

```java
vim /etc/profile  //编辑环境变量文件（CentOS系统默认环境变量文件，其他系统可能不一样）
export PATH=$PATH:/usr/local/rabbitmq_server-3.8.4/sbin //在末尾加入环境变量配置（路径根据实际情况选择）
source /etc/profile //实时生效
```

6. 启动 `RabbitMQ` ，默认端口为 `6752`：

```java
rabbitmq-server -detached  //在后台启动。根据自己实际路径选择，或者也可以选择service或者systemctl等命令启动
```

7. 如果没有报错则说明启动成功，启动之后默认会创建一个 `guest/guest` 账户，只能本地连接，所以还需要再重新创建一个用户，并给新用户授权（当然，我们也可以直接给 `guest` 用户授权）：

```java
rabbitmqctl add_user admin 123456   //创建用户admin
rabbitmqctl set_user_tags admin administrator  //添加标签
rabbitmqctl set_permissions -p / admin ".*" ".*" ".*"  //授权
```

8. `RabbitMQ` 默认还提供了可视化管理界面，需要手动开启一下，默认端口为 `15672`：

```java
rabbitmq-plugins enable rabbitmq_management //启动后台管理系统插件（禁止的话换成disable即可）
```

9. 开启插件之后，可以通过访问：`http://ip:15672/` 访问后台管理系统，并进行一些参数设置，账号密码就是上面添加的 `admin/123456`。

![RabbitMQ后台管理系统](image/1/RabbitMQ后台管理系统.png)

### 安装过程常见错误

安装过程中可能会出现如下图所示错误：

![图1](image/1/安装RabbitMQ缺少依赖.png)

1. odbc:ODBC library - link check failed：

   解决方法：执行命令 `yum install unixODBC.x86_64 unixODBC-devel.x86_64` 进行安装。

2. wx:wxWidgets not found, wx will NOT be usable：

   解决方法：这个属于 `APPLOICATION INFORMATION` ，可以不处理。

3. fakefop to generate placeholder PDF files，documentation: fop is missing.Using fakefop to generate placeholder PDF files：

   解决方法：执行命令 `yum install fop.noarch` 进行安装。

## 利用 Java API 实现一个生产者和消费者

接下来用 `Java` 原生的 `API` 来实现一个简单的生产者和消费者：

1. `pom.xml` 文件引入`RabbitMQ` 客户端依赖：

```xml
 <dependency>
     <groupId>com.rabbitmq</groupId>
     <artifactId>amqp-client</artifactId>
     <version>5.6.0</version>
  </dependency>
```

2. 新建一个消费者 `TestRabbitConsumer` 类：

```java
package com.lonelyWolf.rabbitmq;

import com.rabbitmq.client.*;
import java.io.IOException;

public class TestRabbitConsumer {
    public static void main(String[] args) throws Exception{
        ConnectionFactory factory = new ConnectionFactory();
        factory.setUri("amqp://admin:123456@ip:5672");
        Connection conn = factory.newConnection();//建立连接
       
        Channel channel = conn.createChannel(); //创建消息通道

        channel.queueDeclare("TEST_QUEUE", false, false, false, null);//声明队列
        System.out.println("正在等待接收消息...");

        Consumer consumer = new DefaultConsumer(channel) {//创建消费者
            @Override
            public void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties,byte[] body) throws IOException {
                System.out.println("收到消息: " + new String(body, "UTF-8") + "，当前消息ID为：" + properties.getMessageId());
                System.out.println("收到自定义属性："+ properties.getHeaders().get("name"));
            }
        };
        channel.basicConsume("TEST_QUEUE", true, consumer);//消费之后，回调给consumer
    }
}

```

3. 新建一个生产者 `TestRabbitProducter`类：

```java
package com.lonelyWolf.rabbitmq;

import com.rabbitmq.client.AMQP;
import com.rabbitmq.client.Channel;
import com.rabbitmq.client.Connection;
import com.rabbitmq.client.ConnectionFactory;
import java.util.HashMap;
import java.util.Map;
import java.util.UUID;

public class TestRabbitProducter {
    public static void main(String[] args) throws Exception {
        ConnectionFactory factory = new ConnectionFactory();
        factory.setUri("amqp://admin:123456@ip:5672");
        Connection conn = factory.newConnection();// 建立连接
        
        Channel channel = conn.createChannel();//创建消息通道
        Map<String, Object> headers = new HashMap<String, Object>(1);
        headers.put("name", "双子孤狼");//可以自定义一些自定义的参数和消息一起发送过去

        AMQP.BasicProperties properties = new AMQP.BasicProperties.Builder()
                .contentEncoding("UTF-8") //编码
                .headers(headers) //自定义的属性
                .messageId(String.valueOf(UUID.randomUUID()))//消息id
                .build();

        String msg = "Hello, RabbitMQ";//需要发送的消息
        channel.queueDeclare("TEST_QUEUE", false, false, false, null); //声明队列
        channel.basicPublish("", "TEST_QUEUE", properties, msg.getBytes());//发送消息
        channel.close();
        conn.close();
    }
}
```

4. 先启动消费者，启动之后消费者就会保持和 `RabbitMQ` 的连接，等待消息；然后再运行生产者，消息发送之后，消费者就可以收到消息：

![运行结果](image/1/Java API运行结果.png)

## 利用SpringBoot 实现一个生产者和消费者

接下来再看看 `SpringBoot` 怎么与 `RabbitMQ` 集成并实现一个简单的生产者和消费者：

1. 引入依赖（我这边 `SpringBoot` 用的是 `2.4.0` 版本，所以如果用的低版本这个版本号也需要修改）：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-amqp</artifactId>
    <version>2.4.0</version>
</dependency>
```

2. 新增以下配置文件：

```yaml
spring:
    rabbitmq:
        host: ip
        port: 5672
        username: admin
        password: 123456
```

3. 新建一个配置文件 `RabbitConfig` 类，创建一个队列：

```java
package com.lonely.wolf.rabbit.config;

import org.springframework.amqp.core.Queue;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class RabbitConfig {
    
    @Bean("simpleRabbitQueue")
    public Queue getFirstQueue(){
        Queue queue = new Queue("SIMPLE_QUEUE");
        return queue;
    }
}
```

4. 新建一个消费者 `SimpleConsumer` 类（注意这里监听的名字要和上面定义的保持一致）：

```java
package com.lonely.wolf.rabbit.consumer;

import org.springframework.amqp.rabbit.annotation.RabbitHandler;
import org.springframework.amqp.rabbit.annotation.RabbitListener;
import org.springframework.stereotype.Component;

@RabbitListener(queues = "SIMPLE_QUEUE")
@Component
public class SimpleConsumer {
    @RabbitHandler
    public void process(String msg){
        System.out.println("收到消息：" + msg);
    }
}
```

5. 新建一个消息发送者 `HelloRabbitController` 类（发送消息的队列名要和消费者监听的队列名一致，否则无法收到消息），运行之后调用对应接口，消费者类 `SimpleConsumer` 就可以收到消息：

```java
package com.lonely.wolf.rabbit.controller;

import org.springframework.amqp.core.Message;
import org.springframework.amqp.rabbit.core.RabbitTemplate;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.bind.annotation.RestController;

@RestController
@RequestMapping("/hello")
public class HelloRabbitController {
    @Autowired
    private RabbitTemplate rabbitTemplate;

    @GetMapping(value="/send")
    public String helloSend(@RequestParam(value = "msg",defaultValue = "no message") String msg){
        rabbitTemplate.convertAndSend("SIMPLE_QUEUE",msg);
        return "succ";
    }
}
```

# 总结

本文主要简单讲述了 `MQ` 的发展历史，并介绍了为什么要使用 `MQ` 及 `MQ` 能解决什么问题，紧接着重点介绍了 `AMQP 0.9.1` 模型。掌握了 `AMQP` 模型就基本掌握了 `RabbitMQ` 的工作原理，最后我们通过 `JAVA API` 和 `SpringBoot` 两个例子介绍了如何使用 `RabbitMQ`。