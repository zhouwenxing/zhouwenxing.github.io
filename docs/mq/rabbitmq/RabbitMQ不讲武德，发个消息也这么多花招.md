



[TOC]



# 前言

```properties
本篇博客已被收入GitHub：https://zhouwenxing.github.io/
文中所涉及的源码也已被收入 GitHub：https://github.com/zhouwenxing/lonely-wolf-note (message-mq模块)
```



使用消息队列必须要保证生产者发送的消息能被消费者所接收，那么生产者如何接收消息呢？下图是 `RabbitMQ` 的工作模型：

![AMQP-0-9-1模型](image/1/AMQP-0-9-1.png)

上图中生产者会将消息发送到交换机 `Exchange` 上，再由 `Exchange` 发送给不同的 `Queue` ，而 `Queue` 是用来存储消息队列，那么假如有多个生产者，那么消息发送到交换机 `Exchange` 之后，应该如何和 `Queue` 之间建立绑定关系呢?  

# 如何使用 RabbitMQ 发送消息

`RabbitMQ` 中提供了3种发送消息的路由方式。

## 直连 Direct 模式

通过指定一个精确的绑定键来实现 `Exchange`（交换机） 和 `Queue`（消息队列） 之间的绑定，也就是说，当创建了一个直连类型的交换机时，生产者在发送消息时携带的路由键（routing key），必须与某个绑定键（binding key）**完全匹配**时，这条消息才会从交换机路由到满足路由关系消息队列上，然后消费者根据各自监听的队列就可以获取到消息（如下如吐所示，`Queue1` 绑定了 `order` ，那么这时候发送消息的路由键必须为 `order` 才能分配到 `Queue1` 上）：

![Direct交换机](image/2/direct_exchange.png)

## 主题 Topic 模式

`Direct` 模式会存在一定的局限性，有时候我们需要按类型划分，比如订单类路由到一个队列，产品类路由到另一个队列，所以在 RabbitMQ 中，提供了主题模式来实现模糊匹配。使用主题类型连接方式支持两种通配符：

直连方式只能精确匹配，有时候我们需要实现模糊匹配，那么这时候就需要主题类型的连接方式，在 `RabbitMQ` 中，使用主题类型连接方式支持两种通配符：

- #：表示  `0` 个或者多个单词
- *：表示 `1` 个单词

PS：使用通配符时，单词指的是用英文符号的小数点 `.` 隔开的字符，如：`abc.def` 就表示有 `abc` 和 `def` 两个单词。

下图所示中，因为 `Queue1` 绑定了 `order.#`，所以当发送消息的路由键为 `order` 或者 `order.xxx`时都可以使得消息分配到 `Queue1` 上：  

![Topic交换机](image/2/topic_exchange.png)

## 广播 Fanout 模式

当我们定义了一个广播类型的交换机时就不需要指定绑定键，而且生产者发送消息到交换机上时，也不需要携带路由键，此时当消息到达交换机时，所有与其绑定的队列都会收到消息，这种模式的消息发送适用于消息通知类需求。

如下如所示，`Queue1`，`Queue2`，`Queue3` 三个队列都绑定到了一个 `Fanout` 交换机上，那么当 `Fanout Exchange` 收到消息时，会同时将消息发送给三个队列：

![AMQP-0-9-1模型](image/2/fanout_exchange.png)

在 `RabbitMQ` 提供的后台管理系统中也能查询到创建的交换机和队列等信息，并且可以通过管理后台直接创建队列和交换机：

![管理后台信息查询](image/2/query_exchange.png)

## 消息发送实战

下面通过一个 `SpringBoot` 例子来体会一下三种发送消息的方式。

- 1、`application.yml` 文件中添加如下配置：

```yaml
spring:
  rabbitmq:
    host: ip
    port: 5672
    username: admin
    password: 123456
```

- 2、新增一个 `RabbitConfig` 配置类（为了节省篇幅省略了包名和导入 ），此类中声明了三个交换机和三个队列，并分别进行绑定：

```java
@Configuration
public class RabbitConfig {
    //直连交换机
    @Bean("directExchange")
    public DirectExchange directExchange(){
        return new DirectExchange("LONGLY_WOLF_DIRECT_EXCHANGE");
    }

    //主题交换机
    @Bean("topicExchange")
    public TopicExchange topicExchange(){
        return new TopicExchange("LONGLY_WOLF_TOPIC_EXCHANGE");
    }

    //广播交换机
    @Bean("fanoutExchange")
    public FanoutExchange fanoutExchange(){
        return new FanoutExchange("LONGLY_WOLF_FANOUT_EXCHANGE");
    }


    @Bean("orderQueue")
    public Queue orderQueue(){
        return new Queue("LONGLY_WOLF_ORDER_QUEUE");
    }

    @Bean("userQueue")
    public Queue userQueue(){
        return new Queue("LONGLY_WOLF_USER_QUEUE");
    }

    @Bean("productQueue")
    public Queue productQueue(){
        return new Queue("LONGLY_WOLF_PRODUCT_QUEUE");
    }

    //Direct交换机和orderQueue绑定，绑定键为：order.detail
    @Bean
    public Binding bindDirectExchange(@Qualifier("orderQueue") Queue queue, @Qualifier("directExchange") DirectExchange directExchange){
        return BindingBuilder.bind(queue).to(directExchange).with("order.detail");
    }

    //Topic交换机和userQueue绑定，绑定键为：user.#
    @Bean
    public Binding bindTopicExchange(@Qualifier("userQueue") Queue queue, @Qualifier("topicExchange") TopicExchange topicExchange){
        return BindingBuilder.bind(queue).to(topicExchange).with("user.#");
    }

    //Fanout交换机和productQueue绑定
    @Bean
    public Binding bindFanoutExchange(@Qualifier("productQueue") Queue queue, @Qualifier("fanoutExchange") FanoutExchange fanoutExchange){
        return BindingBuilder.bind(queue).to(fanoutExchange);
    }
}
```

- 3、新建一个消费者 `ExchangeConsumer` 类，不同的方法实现分别监听不同的队列：

```java
@Component
public class ExchangeConsumer {

    /**
     * 监听绑定了direct交换机的的消息队列
     */
    @RabbitHandler
    @RabbitListener(queues = "LONGLY_WOLF_ORDER_QUEUE")
    public void directConsumer(String msg){
        System.out.println("direct交换机收到消息：" + msg);
    }

    /**
     * 监听绑定了topic交换机的的消息队列
     */
    @RabbitHandler
    @RabbitListener(queues = "LONGLY_WOLF_USER_QUEUE")
    public void topicConsumer(String msg){
        System.out.println("topic交换机收到消息：" + msg);
    }

    /**
     * 监听绑定了fanout交换机的的消息队列
     */
    @RabbitHandler
    @RabbitListener(queues = "LONGLY_WOLF_PRODUCT_QUEUE")
    public void fanoutConsumer(String msg){
        System.out.println("fanout交换机收到消息：" + msg);
    }
}
```

- 4、新增一个 `RabbitExchangeController` 类来作为生产者，进行消息发送：

```java
@RestController
@RequestMapping("/exchange")
public class RabbitExchangeController {
    @Autowired
    private RabbitTemplate rabbitTemplate;

    @GetMapping(value="/send/direct")
    public String sendDirect(String routingKey,@RequestParam(value = "msg",defaultValue = "no direct message") String msg){
        rabbitTemplate.convertAndSend("LONGLY_WOLF_DIRECT_EXCHANGE",routingKey,msg);
        return "succ";
    }
    @GetMapping(value="/send/topic")
    public String sendTopic(String routingKey,@RequestParam(value = "msg",defaultValue = "no topic message") String msg){
        rabbitTemplate.convertAndSend("LONGLY_WOLF_TOPIC_EXCHANGE",routingKey,msg);
        return "succ";
    }
    @GetMapping(value="/send/fanout")
    public String sendFaout(String routingKey,@RequestParam(value = "msg",defaultValue = "no faout message") String msg){
        rabbitTemplate.convertAndSend("LONGLY_WOLF_FANOUT_EXCHANGE",routingKey,msg);
        return "succ";
    }
}

```

- 5、启动服务，当我们调用第一个接口时候，路由键和绑定键 `order.detail` 精确匹配时，`directConsumer` 就会收到消息，同样的，调用第二接口时，路由键满足 `user.#` 时，`topicConsumer` 就会收到消息，而只要调用第三个接口，不论是否指定路由键，`fanoutConsumer` 都会收到消息。

# 消息过期了怎么办

简单的发送消息我们学会了，难道这就能让我们就此止步了吗？显然是不能的，要玩就要玩高级点，所以接下来让我们给消息加点佐料。

## TTL（Time-To-Live）

`TTL` 即 一条消息在队列中的最大存活时间。在一条在队列中超过配置的 `TTL` 的消息称为已死消息。但是需要注意的是，已死消息并不能保证会立即从队列中删除，但是能保证已死的消息不会被投递出去。

设置 `TTL` 的方式有两种：

- 1、给队列设置 `x-message-ttl`，此时所有被投递到队列中的消息，都会在到达 `TTL` 时成为已死消息。

  这种情况就会出现当一条消息同时路由到 `N` 个带有 `TTL` 时间的队列，而由于每个队列的 `TTL` 不一定相同，所以同一条消息在不同的队列中可能会在不同时间死亡或者不会死亡（未设置 `TTL` ），所以一个队列中的消息死亡不会影响到其他队列中的消息。  

- 2、单独给某一条消息设置过期时间。

  此时需要注意的时，当消息达到 `TTL` 时，可能不会马上被丢弃，因为只有处于队列头部消息过期后才会被丢弃，假如队列头部的消息没有设置 `TTL`，而第 `2` 条消息设置了 `TTL`，那么即使第 `2` 条消息成为了已死消息，也必须要等到队列头部的消息被消费之后才会被丢弃，而已死消息在被丢弃之前也会被计入统计数据（比如队列中的消息总数）。所以为了更好的利用 `TTL` 特性，建议让消费者在线消费消息，这样才能确保消息更快的被丢弃，防止消息堆积。 

PS：消息过期和消费者传递之间可能存在自然的竞争条件。例如，消息可能在发送途中（未到达消费者）过期。

## 队列的生存

和 `TTL` 针对消息不同的是，我们可以通过设置过期时间属性 `x-expires`` 来处理队列，当在指定过期时间内内未使用队列时，服务器保证将删除队列（但是无法保证在过期时间过后队列将以多快的速度被删除）。

## TTL 和过期时间实战

- 1、在上面定义的 `RabbitConfig` 类中，再新增一个 `TTL` 队列并将其绑定到 `direct` 交换机上：

```java
@Bean("ttlQueue")
public Queue ttlQueue(){
    Map<String, Object> map = new HashMap<String, Object>();
    map.put("x-message-ttl", 5000);//队列中所有消息5秒后过期
    map.put("x-expires", 100000);//队列闲置10秒后被删除
    //参数1-name：队列名称
    //参数2-durable：是否持久化
    //参数3-exclusive:是否排他。设置为true时，则该队列只对声明当前队列的连接(Connection)可用,一旦连接断开，队列自动被删除
    //参数4-autoDelete：是否自动删除。前提是必须要至少有一个消费者先连上当前队列，然后当所有消费者都断开连接之后，队列自动被删除
    return new Queue("LONGLY_WOLF_TTL_QUEUE",false,false,false,map);
    }

//ttl队列绑定到direct交换机（交换机和队列可以多对多）
@Bean
public Binding ttlBindFanoutExchange(@Qualifier("ttlQueue") Queue queue, @Qualifier("directExchange") DirectExchange directExchange){
    return BindingBuilder.bind(queue).to(directExchange).with("test.ttl");
}
```

- 2、在 `ExchangeConsumer` 消费者类上监听 `TTL` 队列（和其他消费者不同的时候，这里为了打印出队列属性，改成了通过 `Message` 对象来接收消息 ）：

```java
/**
 * 监听ttl消息队列
*/
@RabbitHandler
@RabbitListener(queues = "LONGLY_WOLF_TTL_QUEUE")
public void ttlConsumer(Message message){
    System.out.println("ttl队列收到消息：" + new String(message.getBody()));
    System.out.println("ttl队列收到消息：" + JSONObject.toJSONString(message.getMessageProperties()));
}
```

- 3、在生产者类 `RabbitExchangeController` 上新增一个接口用来测试发送过期消息，这里通过 `MessageProperties` 设置的 `expiration` 属性就相当于是给单条消息设置了一个 `TTL` ：

```java
@GetMapping(value="/send/ttl")
public String sendTtl(String routingKey,@RequestParam(value = "msg",defaultValue = "no ttl message") String msg){
    MessageProperties messageProperties = new MessageProperties();
    messageProperties.setExpiration("5000");//5秒后被删除，即TTL属性（针对单条消息）
    Message message = new Message(msg.getBytes(), messageProperties);
    rabbitTemplate.convertAndSend("LONGLY_WOLF_DIRECT_EXCHANGE",routingKey,message);
    return "succ";
}
```

- 4、此时如果我们把消费者的监听去掉之后再发送消息，在管理后台就可以看到 `5` 秒之后消息会被删除，`10` 秒之后队列会被删除。

PS：如果同时给队列和单条消息都设置了 `TTL`，则会以时间短的为主。 

## 其他属性

队列中还有其他一些属性可以设置，在这里我们就不一一举例了：

- x-message-ttl：队列中消息的存活时间（毫秒），达到TTL的消息可能会被删除。
- x-expires：队列在多长时间（毫秒）没有被访问以后会被删除。
- x-max-length：队列中的最大消息数。
- x-max-length-bytes：队列的最大容量（bytes）。
- overflow：队列溢出之后的策略。主要可以配置如下参数：`reject-publish` - 直接丢弃最近发布的消息，如若启用了` publisher confirm`（发布者确认），发布者将通过发送 `basic.nack` 消息通知拒绝，如果当前队列绑定有多个消费者，则消息在收到 `basic.nack` 拒绝通知后，仍然会被发布到其他队列；`drop-head` - 丢弃队列头部消息（集群模式下只支持这种策略） `reject-publish-dlx` - 最近发布的消息会进入死信队列。
- x-dead-letter-exchange：队列的死信交换机。
- x-dead-letter-routing-key：死信交换机的路由键。
- x-single-active-consumer：true/false。表示是否最多只允许一个消费者消费，如果有多个消费者同时绑定，则只会激活第一个，除非第一个消费者被取消或者死亡，才会自动转到下一个消费者。
- x-max-priority：队列中消息的最大优先级， 消息的优先级不能超过它。
- x-queue-mode：`3.6.0` 版本引入的，主要是为了实现惰性加载。队列将收到的消息尽可能快的进行持久化操作到磁盘上，然后只有在用户请求的时候才会加载到 `RAM` 内存。这个参数支持两个值：`default` 和 `lazy`。当不进行设置的时候，就是默认为 `default`，不做任何改变；当设置为 `lazy` 就会进行懒加载。 
- x-queue-master-locator：为了保证消息的 `FIFO`，所以在高可用集群模式下需要选择一个节点作为主节点。这个参数主要有三种模式：`min-masters`- 托管最小数量的绑定主机的节点；`client-local`- 选择声明的队列已经连接到客户端的节点；`random`- 随机选择一个节点。

# 神奇的死信队列（Dead Letter）

上面的参数介绍中，提到了死信队列，这又是什么新鲜的东西呢？其实从名字上来看很好理解，就是指的已死的消息，或者说无家可归的消息。一个消息进入死信队列，主要有以下三种条件：

- 1、消息被消费者拒绝并且未设置重回队列。

- 2、消息过期（即设置了 `TTL`）。 

- 3、队列达到最大长度，超过了 `Max length` 或 `Max length bytes`，则队列头部的消息会被发送到死信队列。  

## 死信队列实战

- 1、在上面定义的 `RabbitConfig` 类中，定义一个死信交换机，并将之前的 `ttl` 队列新增一个属性 `x-dead-letter-exchange`，最后再将死信队列和死信交换机进行绑定：

```java
//直连死信交换机(也可以用topic或者fanout类型交换机)
@Bean("deatLetterExchange")
public DirectExchange deatLetterExchange(){
    return new DirectExchange("LONGLY_WOLF_DEAD_LETTER_DIRECT_EXCHANGE");
}
@Bean("ttlQueue")
public Queue ttlQueue(){
    Map<String, Object> map = new HashMap<String, Object>();
    map.put("x-message-ttl", 5000);//队列中所有消息5秒后过期
    map.put("x-dead-letter-exchange", "LONGLY_WOLF_DEAD_LETTER_DIRECT_EXCHANGE");//已死消息会进入死信交换机
    return new Queue("LONGLY_WOLF_TTL_QUEUE",false,false,false,map);
}
//死信队列
@Bean("deadLetterQueue")
public Queue deadLetterQueue(){
    return new Queue("LONGLY_WOLF_DEAD_LETTER_QUEUE");
}
```

- 2、在 `ExchangeConsumer` 消费者类上将监听 `TTL` 队列的监听取消，注释掉监听：

```java
	/**
     * 监听ttl消息队列
     */
    @RabbitHandler
//    @RabbitListener(queues = "LONGLY_WOLF_TTL_QUEUE")
    public void ttlConsumer(Message message){
        System.out.println("ttl队列收到消息：" + new String(message.getBody()));
        System.out.println("ttl队列收到消息：" + JSONObject.toJSONString(message.getMessageProperties()));
    }
```

- 3、此时 `TTL` 队列无消费者，并且设置了消息的 `TTL` 为 `5` 秒，所以 `5` 秒之后就会进入死信队列。
- 5、访问接口：`http://localhost:8080/exchange/send/ttl?routingKey=test&msg=测试死信队列`，发送消息之后，等待  `5` 秒就查看消息，进入死信队列：

![消息进入死信队列](image/2/dead_letter.png)

# 消息真的发送成功了吗

了解了消息的基本发送功能之后，就可以高枕无忧了吗？消息发出去之后，消费者真的收到消息了吗？消息发送之后如何知道消息发送成功了？假如发送消息路由错了导致无法路由到队列怎么办？大家是不是都有这些疑问呢？别着急，接下来就让我们来一一来分析一下。

一条消息从生产者开始发送消息到消费者消费完消息主要可以分为以下 `4` 个阶段：

- 1、生产者将消息发送到 `Broker` （即：`RabbitMQ` 的交换机）。
- 2、交换机将消息路由到队列。
- 3、队列收到消息后存储消息。
- 4、消费者从队列获取消息进行消费。

接下来我们就从这 `4` 个步骤上来逐步分析 `RabbitMQ` 如何保证消息发送的可靠性。

## 消息真的到达交换机了吗

当我们发送一条消息之后，如何知道对方收到消息了？这就和我们写信一样，写一封信出去，如何知道对方收到我们寄出去的信？最简单的方式就是对方也给我们回一封信，我们收到对方的回信之后就可以知道自己的信已经成功寄达。

在 `RabbitMQ` 中服务端也提供了 `2` 种方式来告诉客户端（生产者）是否收到消息：`Transaction`（事务）模式和 `Confirm`（确认）模式。

### `Transaction`（事务） 模式 

在 `Java API` 编程中开启事务只需要增加以下代码即可：

```java
 try {
     channel.txSelect();//开启事务
     channel.basicPublish("", QUEUE_NAME, null, msg.getBytes());
     channel.txCommit();//提交事务
 }catch (Exception e){
     channel.txRollback();//消息回滚
 }
```

在 `Spring Boot` 中需要对 `RabbitTemplate` 进行事务设置：

```java
@Bean
public RabbitTemplate rabbitTemplate(ConnectionFactory connectionFactory){
    RabbitTemplate rabbitTemplate = new RabbitTemplate();
    rabbitTemplate.setConnectionFactory(connectionFactory);
    rabbitTemplate.setChannelTransacted(true);//开启事务
    return rabbitTemplate;
}
```

为了了解 `RabbitMQ` 当中事务机制的原理，我们在 `Wireshark` 中输入 `ip.addr==192.168.1.1` 对本地 `ip` 进行抓包，发送一条消息之后，抓到如下数据包：

![事务机制抓包](image/2/transaction.png)

通过数据包，可以得出开启事务之后，除了原本的发送消息之外，多出了开启事务和事务提交的通信：

![事务机制抓包](image/2/transaction-bpm.png)

开启事务之后，有一个致命的缺点就是**发送消息流程会被阻塞**。也就是说必须一条消息发送成功之后，才会允许发送另一条消息。正因为事务模式有这个缺点，所以一般情况下并不建议在生产环境开启事务，那么有没有更好的方式来实现消息的送达确认呢？那么就让我们再看看`Confirm`（确认）模式。

### `Confirm`（确认）模式

消息确认模式又可以分为三种（**事务模式和确认模式无法同时开启**）：

- 单条确认模式：发送一条消息，确认一条消息。此种确认模式的效率也不高。
- 批量确认模式：发送一批消息，然后同时确认。批量发送有一个缺点就是同一批消息一旦有一条消息发送失败，就会收到失败的通知，需要将这一批消息全部重发。
- 异步确认模式：一边发送一边确认，消息可能被单条确认也可能会被批量确认。

#### Java API 实现确认模式

- 单条消息确认模式

```java
channel.confirmSelect();//开启确认模式
channel.basicPublish("",QUEUE_NAME,null,msg.getBytes());
if (channel.waitForConfirms()){//wait.ForConfirms(long time)方法可以指定等待时间
    System.out.println("消息确认发送成功");
}
```

- 批量确认模式

```java
channel.confirmSelect();//开启确认模式
//批量发送
for (int i=0;i<10;i++){
    channel.basicPublish("",QUEUE_NAME,null,msg.getBytes());
}
try{
    channel.waitForConfirmsOrDie();
}catch (IOException e){//只要有1条消息未被确认，就会抛出异常
    System.out.println("有消息发送失败了");
}
```

- 异步确认模式

```java
channel.addConfirmListener(new ConfirmListener() {
    /**
      * 已确认消息，即发送成功后回调
      * @param deliveryTag -唯一标识id（即发送消息时获取到的nextPublishSeqNo）
      * @param multiple - 是否批量确认，当multiple=true，表示<=deliveryTag的消息被批量确认，multiple=false，表示只确认了单条
      */
    @Override
    public void handleAck(long deliveryTag, boolean multiple) throws IOException {//成功回调
        System.out.println("收到确认消息了");
        //TODO 可以做一些想做的事
    }

    /**
       * 发送失败消息后回调
       * @param deliveryTag -唯一标识id（即发送消息时获取到的nextPublishSeqNo）
       * @param multiple - 是否批量确认，当multiple=true，表示<=deliveryTag的消息被批量确认，multiple=false，表示只确认了单条
       */
    @Override
    public void handleNack(long deliveryTag, boolean multiple) throws IOException {//失败回调
        if (multiple) {//批量确认，<deliveryTag的消息都发送失败
            //TODO 消息重发？
        } else {//非批量，=deliveryTag的消息发送失败
            //TODO 消息重发？
        }
    }
});

channel.confirmSelect();//开启确认模式
for (int i=0;i<10;i++){//批量发送
    long nextSeqNo = channel.getNextPublishSeqNo();//获取发送消息的唯一标识(从1开始递增)
    //TODO 可以考虑把消息id存起来
    channel.basicPublish("",QUEUE_NAME,null,msg.getBytes());
}
```

#### SpringBoot 实现确认模式

通过配置文件 `spring.rabbitmq.publisher-confirm-type` 参数进行配置确认（旧版本是 `spring.rabbitmq.publisher-confirms` 参数）。

- 1、新增配置文件属性配置

```yaml
spring:
  rabbitmq:
    publisher-confirm-type: correlated # none-表示禁用回调(默认) simple- 参考RabbitExchangeController#sendWithSimpleConfirm()方法
```

- 2、`RabbitConfig` 配置文件中修改如下：

```java
 @Bean
    public RabbitTemplate rabbitTemplate(ConnectionFactory connectionFactory){
        RabbitTemplate rabbitTemplate = new RabbitTemplate();
        rabbitTemplate.setConnectionFactory(connectionFactory);
//        rabbitTemplate.setChannelTransacted(true);//开启事务
        //消息是否成功发送到Exchange
        rabbitTemplate.setConfirmCallback(new RabbitTemplate.ConfirmCallback() {
            @Override
            public void confirm(CorrelationData correlationData, boolean ack, String cause) {
                if (!ack){//消息发送失败
                    System.out.println("消息发送失败，原因为：" + cause);
                    return;
                }
                //消息发送成功
                System.out.println("消息发送成功");
            }
        });
        return rabbitTemplate;
    }
```

这样当我们发送消息成功之后，就会收到回调。

- 3、当上面的参数配置修改为 `simple`，则需要在发送消息的时候使用 `invoke` 调用 `waitForConfirms` 或者 `waitForConfirmsOrDie` 方法来确认是否发送成功：

```java
 @GetMapping(value="/send/confirm")
 public String sendWithSimpleConfirm(String routingKey,@RequestParam(value = "msg",defaultValue = "no direct message") String msg){
       //使用waitForConfirms方法确认
        boolean sendFlag = rabbitTemplate.invoke(operations -> {
            rabbitTemplate.convertAndSend(
                    "LONGLY_WOLF_DIRECT_EXCHANGE",
                    "routingKey",
                    msg
            );
            return rabbitTemplate.waitForConfirms(5000);
        });
        //也可以使用waitForConfirmsOrDie方法确认
        boolean sendFlag2 = rabbitTemplate.invoke(operations -> {
            rabbitTemplate.convertAndSend(
                    "LONGLY_WOLF_DIRECT_EXCHANGE",
                    "routingKey",
                    msg
            );
            try {
                rabbitTemplate.waitForConfirmsOrDie(5000);
            }catch (Exception e){
                return false;
            }
            return true;
        });
        System.out.println(sendFlag);
        System.out.println(sendFlag2);
        return "succ";
    }
```

## 消息无法从交换机路由到正确的队列怎么办

上面通过事务或者确认机制确保了消息成功发送到交换机，那么接下来交换机会负责将消息路由到队列，这时候假如队列不存在或者路由错误就会导致消息路由失败，这又该如何保证呢？

同样的，`RabbitMQ` 中也提供了 `2` 种方式来确保消息可以正确路由到队列：开启监听模式或者通过新增备份交换机模式来备份数据。

### 监听回调

上面介绍的是消息是否发送到交换机的回调，而从交换机路由到队列，同样可以开启确认模式。

#### Java API 方式开启监听模式

下面就是开启监听主要代码，为了节省篇幅，省略了其余不相干代码（完成代码已上传至 `GitHub`）

```java
channel.addReturnListener(new ReturnListener() {
     @Override
     public void handleReturn(int replyCode, String replyText, String exchange, String routingKey, AMQP.BasicProperties properties, byte[] body) throws IOException {
         System.out.println("收到未路由到队列的回调消息：" + new String(body));
     }
 });
//注意这里的第三个参数，mandatory需要设置为true（发送一个错误的路由，即可收到回调）
channel.basicPublish(EXCHANGE_NAME,"ERROR_ROUTING_KEY",true,null,msg.getBytes());
```

#### Spring Boot 开启监听模式

在 `RabitConfig` 类中添加如下配置：

```java
 @Bean
public RabbitTemplate rabbitTemplate(ConnectionFactory connectionFactory){
    RabbitTemplate rabbitTemplate = new RabbitTemplate();
    rabbitTemplate.setConnectionFactory(connectionFactory);

    rabbitTemplate.setMandatory(true);//开启监听回调
    //消息是否成功被路由到队列，没有路由到队列时会收到回调（原setReturnCallback在2.0版本已过期）
    rabbitTemplate.setReturnsCallback(new RabbitTemplate.ReturnsCallback() {
        @Override
        public void returnedMessage(ReturnedMessage returnedMessage) {
            System.out.println("收到未路由到队列的回调消息：" + new String(returnedMessage.getMessage().getBody()));
        }
    });
    return rabbitTemplate;
}
```

### 备份交换机

除了开启监听的方式，还可以通过定义备份交换机的方式来实现，当原交换机无法正确路由到队列时，则会进入备份交换机，再由备份交换机路由到正确队列（要注意区分备份交换机和死信交换机的区别）。

#### Java API 实现备份交换机

下面就是一个实现备份交换机的例子，因为这里备份交换机定义的是 `Topic` 类型，所有路由必须满足定义好的路由，实际使用中一般会设置会 `Fanout`，因为无法预测错误的路由到底是多少：

```java
 //声明交换机且指定备份交换机
Map<String,Object> argMap = new HashMap<String,Object>();
argMap.put("alternate-exchange","TEST_ALTERNATE_EXCHANGE");
channel.exchangeDeclare(EXCHANGE_NAME, BuiltinExchangeType.DIRECT,false,false,argMap);
//队列和交换机进行绑定
channel.queueBind(QUEUE_NAME,EXCHANGE_NAME,ROUTEING_KEY);

//声明备份交换机和备份队列，并绑定（为了防止收不到消息，备份交换机一般建议设置为Fanout类型）
channel.queueDeclare("BAK_QUEUE", false, false, false, null);
channel.exchangeDeclare("TEST_ALTERNATE_EXCHANGE", BuiltinExchangeType.TOPIC);
channel.queueBind("BAK_QUEUE","TEST_ALTERNATE_EXCHANGE","ERROR.#");

String msg = "I'm a bak exchange msg";
channel.basicPublish(EXCHANGE_NAME,"ERROR.ROUTING_KEY",null,msg.getBytes());
```

#### Spring Boot 实现备份交换机

`Spring Boot` 实现备份交换机原理和 `Java API` 实现相同：

- 1、首先在 `RabbiConfig` 中新增两个交换机，一个是原始交换机，一个是备份交换机，同时新增一个备份队列和备份交换机进行绑定，这里的备份交换机是一个 `Fanout` 类型，注意因为这里主要是演示备份交换机，所以这里的原始交换机没有和任何队列绑定，也就无法路由到队列，从而使得消息进入备份交换机：

```java
//用于测试备份交换机的原直连交换机
@Bean("bakDirectEchange")
public DirectExchange bakDirectEchange(){
    Map argMap = new HashMap<>();
    argMap.put("alternate-exchange", "LONGLY_WOLF_BAK_FANOUT_EXCHANGE");
    return new DirectExchange("LONGLY_WOLF_BAK_ORIGIN_DIRECT_EXCHANGE",false,false,argMap);
}

//备份广播交换机
@Bean("bakFanoutExchange")
public FanoutExchange bakFanoutExchange(){
    return new FanoutExchange("LONGLY_WOLF_BAK_FANOUT_EXCHANGE");
}
//备份队列
@Bean("bakQueue")
public Queue bakQueue(){
    return new Queue("LONELY_WOLF_BAK_QUEUE");
}
//备份交换机和备份队列进行绑定
@Bean
public Binding BindExchange(@Qualifier("bakQueue") Queue queue, @Qualifier("bakFanoutExchange") FanoutExchange fanoutExchange){
    return BindingBuilder.bind(queue).to(fanoutExchange);
}
```

2、在消费者类 `ExchangeConsumer` 中监听备份队列：

```java
 /**
  * 监听备份消息队列
  */
@RabbitHandler
@RabbitListener(queues = "LONELY_WOLF_BAK_QUEUE")
public void bakQueueConsumer(Message message){
    System.out.println("备份队列收到消息：" + new String(message.getBody()));
}
```

- 3、最后在生产者类 `RabbitExchangeController` 中新增一个消息发送的方法进行消息发送：

```java
@GetMapping(value="/send/bak")
public String sendBak(String routingKey,@RequestParam(value = "msg",defaultValue = "no bak message") String msg){
    rabbitTemplate.convertAndSend("LONGLY_WOLF_BAK_ORIGIN_DIRECT_EXCHANGE",routingKey,msg);
    return "succ";
}
```

调用之后可以看到，备份队列会收到消息，从而说明了消息在无法路由到队列时会进入到备份队列。

## 队列存储消息后发生异常怎么办

在保证了前面两个阶段的可靠性之后，消息终于安全抵达了队列，那么这时候就绝对安全了吗？

当我们的消费者的消费速度跟不上生产者的生产速度时，就会导致消息堆积在队列中，而默认消息是没有持久化的，存在于内存之中，所以假如服务器宕机等故障发生，就会导致队列中的数据丢失。

这里的解决方案也很简单，就是将消息进行持久化，在 `RabbitMQ` 当中，持久化也可以分为 `3` 种：交换机持久化，队列持久化和消息持久化。

虽然说持久化能一定程度上保证消息的可靠性，然而当出现了服务器的磁盘损坏，依然可能出现消息丢失，所以为了更加完美，`RabbitMQ` 集群可能是必须的，当然，本文不会涉及到集群的知识，集群的知识以及搭建会放到下次再来分析。

### 交换机持久化

声明交换机时，`durable` 参数设置为 `true`。

### 队列持久化

声明队列时，`durable` 参数设置为 `true`。

### 消息持久化

发送消息时可以将消息设置为持久化。

#### Java API 消息持久化

 在 `Java API` 中，可以通过如下方式设置消息持久化：

```java
//deliveryMode=2表示消息持久化
AMQP.BasicProperties properties = new AMQP.BasicProperties.Builder().deliveryMode(2).build();
channel.basicPublish("exchangeName","routingKey",properties,msg.getBytes());
```

#### Spring Boot 消息持久化

在 `Spring Boot` 中可以通过如下方式将消息设置为持久化：

```java
MessageProperties messageProperties = new MessageProperties();
messageProperties.setDeliveryMode(MessageDeliveryMode.PERSISTENT);//消息持久化
Message message = new Message(msg.getBytes(), messageProperties);
rabbitTemplate.convertAndSend("exchangeName","routingKey",message);
```

## 消费者消费消息失败了怎么办

踏遍千山万水，经过 `3` 层地狱模式，消息终于被消费者拿到手了，然而悲剧的事情又发生了，消费者消费消息的时候可能因为消费者本身的问题或者其他意外导致了消费者消费消息失败了，这时候消息还是没能被正确处理，这时候难道眼睁睁看着最后关头了束手无策了吗？

非也，作为一款如此优秀的消息队列，怎么可能没考虑到这种场景呢。还记不记得上面我们提到的确认模式，实际上，上面的两种确认模式都属于服务端的确认，在 `RabbitMQ` 中为消费者也提供了确认模式，这就是消费者的确认。

### 消费者确认（ack）

队列当中会把消息删除的前提就是这条消息被消费者消费掉了，但是服务器如何知道消息被消费了呢？这就是需要通过消费者确认之后才会删除，而我们前面在介绍消息发送的时候貌似并没有看到消费者确认流程，这是因为消费者默认在收到消息后会给服务器一个应答，服务端收到消费者的应答之后，就会删除消息。

#### Java API 实现消费者应答

在 `Java API` 中应答方式有两种，自动应答和手动应答，当自动应答时，则只要消费者收到消息就会给服务端确认，不在乎消息是否消费成功。

- 1、新建一个消费者 `AckConsumer` 类（省略了包名和导入），这里为了实现方便，通过生产者的头部标记来决定采用何种应答策略：

```java
public class AckConsumer {
    private static String QUEUE_NAME = "ACK_QUEUE";
    public static void main(String[] args) throws Exception{
        //1.声明连接
        ConnectionFactory factory = new ConnectionFactory();
        factory.setUri("amqp://username:password@ip:port");

        //2.建立连接
        Connection conn = factory.newConnection();
        //3.创建消息通道
        Channel channel = conn.createChannel();
        //4.声明队列（默认交换机AMQP default，Direct）
        channel.queueDeclare(QUEUE_NAME, false, false, false, null);
        System.out.println(" 等待接收消息...");

        // 创建消费者
        Consumer consumer = new DefaultConsumer(channel) {
            @Override
            public void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties,
                                       byte[] body) throws IOException {
                System.out.println("收到消息: " + new String(body, "UTF-8"));
                Map<String,Object> map = properties.getHeaders();//获取头部消息
                String ackType = map.get("ackType").toString();
                if (ackType.equals("ack")){//手动应答
                    channel.basicAck(envelope.getDeliveryTag(),true);
                }else if(ackType.equals("reject-single")){//拒绝单条消息
                    //拒绝消息。requeue参数表示消息是否重新入队
                    channel.basicReject(envelope.getDeliveryTag(),false);
                    //                    channel.basicNack(envelope.getDeliveryTag(),false,false);
                }else if (ackType.equals("reject-multiple")){//拒绝多条消息
                    //拒绝消息。multiple参数表示是否批量拒绝，为true则表示<deliveryTag的消息都被拒绝
                    channel.basicNack(envelope.getDeliveryTag(),true,false);
                }
            }
        };

        //开始获取消息,第二个参数 autoAck表示是否开启自动应答
        channel.basicConsume(QUEUE_NAME, false, consumer);
    }
}

```

- 2、新建一个生产者 `AckProducer` 类（省略了包名和导入）：

```java
public class AckProducter {
    private static String QUEUE_NAME = "ACK_QUEUE";//队列
    private static String EXCHANGE_NAME = "ACK_EXCHANGE";//交换机
    private static String ROUTEING_KEY = "test";
    public static void main(String[] args) throws Exception {
        ConnectionFactory factory = new ConnectionFactory();
        factory.setUri("amqp://admin:123456@47.107.155.197:5672");
        // 建立连接
        Connection conn = factory.newConnection();
        // 创建消息通道
        Channel channel = conn.createChannel();
        Map<String, Object> headers = new HashMap<String, Object>(1);
        headers.put("ackType", "ack");//请应答
//        headers.put("ackType", "reject-single");//请单条拒绝
//        headers.put("ackType", "reject-multiple");//请多条拒绝

        AMQP.BasicProperties properties = new AMQP.BasicProperties.Builder()
                .contentEncoding("UTF-8")  // 编码
                .headers(headers) // 自定义属性
                .messageId(String.valueOf(UUID.randomUUID()))
                .build();

        String msg = "I'm a ack message";
        //声明队列
        channel.queueDeclare(QUEUE_NAME, false, false, false, null);
        //声明交换机
        channel.exchangeDeclare(EXCHANGE_NAME, BuiltinExchangeType.DIRECT,false);
        //队列和交换机进行绑定
        channel.queueBind(QUEUE_NAME,EXCHANGE_NAME,ROUTEING_KEY);
        // 发送消息
        channel.basicPublish(EXCHANGE_NAME, ROUTEING_KEY, properties, msg.getBytes());

        channel.close();
        conn.close();
    }
}
```



#### Spring Boot 实现消费者应答

在 `Spring Boot` 中消费者给服务端的确认方式分为 `3` 种：

- `NONE`：自动应答（`ack`）。
- `MANUAL`：手动应答（`ack`）。如果设置为手动应答，而消费者又迟迟不给服务器应答，那么消息就会一直存在队列，可能会造成消息堆积和重复消费现象。
- `AUTO`：当没有抛出异常时会自动应答（`ack`）。除此外，当发生异常时，分为以下三种情况：
  - 1、当抛出 `AmqpRejectAndDontRequeueException` 异常时，消息会被拒绝，也不会重新入队。
  - 2、当抛出 `ImmediateAcknowledgeAmqpException` 异常时，消费者会自动发送应答给服务端。
  - 3、当抛出其他异常时，消息会被拒绝，且会重新入队。当出现这种情况且消费者只有一个时，非常容易造成死循环，所以应该极力避免这种情况的发生。

- 1、`Spring Boot` 中可以通过参数控制应答类型：

```yaml
spring:
  rabbitmq:
    listener:
      type: simple # direct类型是2.0之后才有的
      simple:
        acknowledge-mode: manual
```

- 2、在消费者类 `ExchangeConsumer` 中新建一个方法来监听队列，其中第一个注释掉的方法是原本存在的，第二个方法是新增的，主要新增了几个参数，注意 `Channel` 是 `com.rabbitmq.client.Channel` 包下的：

```java
/**
 * 监听绑定了direct交换机的的消息队列
 */
//    @RabbitHandler
//    @RabbitListener(queues = "LONGLY_WOLF_ORDER_QUEUE")
//    public void directConsumer(String msg){
//        System.out.println("direct交换机收到消息：" + msg);
//    }

/**
 * 监听绑定了direct交换机的的消息队列，并进行手动应答
 */
@RabbitHandler
@RabbitListener(queues = "LONGLY_WOLF_ORDER_QUEUE")
public void manualDirectConsumer(String msg, Channel channel,Message message) throws IOException {
    System.out.println("direct交换机收到消息：" + msg + "。此消息需要手动应答");
    channel.basicAck(message.getMessageProperties().getDeliveryTag(),false);//手动应答
}
```

- 3、或者也可以通过 `SimpleMessageListenerContainer` 类实现监听，新建一个 `RabbitAckConfig` 类（省略了包名和导入）：

```java
@Configuration
public class RabbitAckConfig {
    @Bean
    public SimpleMessageListenerContainer messageListenerContainer(ConnectionFactory connectionFactory){
        SimpleMessageListenerContainer container = new SimpleMessageListenerContainer();
        container.setConnectionFactory(connectionFactory);
        container.setQueueNames("LONGLY_WOLF_ORDER_QUEUE");//设置监听队列名
        container.setAcknowledgeMode(AcknowledgeMode.MANUAL);//手动确认
        container.setMessageListener((ChannelAwareMessageListener) (message, channel) -> {//消息处理
            System.out.println("收到消息：" + new String(message.getBody()) + "。此消息需要手动应答");
            channel.basicAck(message.getMessageProperties().getDeliveryTag(),false);
        });
        return container;
    }
}
```

PS：需要注意的是，这两种方式不要同时使用，否则无法保证消息会被哪个监听到。

# 仅靠 RabbitMQ 自身可靠性能实现业务需求吗

上面介绍的两种确认模式，服务端确认和消费者确认。其中服务端确认是会回调给生产者的，所以生产者可以知道消息是否已经到达服务器且是否正确路由到队列，然而，对于消费者的确认，生产者是不知道的，这是因为消息队列的作用之一就是为了实现生产者和消费者的解耦，换言之，**消费者知道消息成功发送到队列，但是无法知道消息是否被消费者消费**。

所以为了知道消息是否被成功消费，主要有两种思路：

- 1、消费者在消费成功之后需要回调生产者提供的API来告知消息已经被消费
- 2、服务端在收到消费者确认后给生产者一个回执通知

然而假如生产者迟迟没有收到消费者是否消费成功的信息，那么可能就需要补偿，比如微信支付等都会有补偿机制，间隔一定时间就将消息重发一次。

补偿机制同时也会带来一个问题，假如说消费者消费成功了，但是在告诉生产者的时候失败了，那么这时候消息如果再次补偿就会造成重复消费，所以消费者需要支持幂等（即无论一条消息被消费多少次，都不会改变结果）。当然，同时还有其他场景需要考虑，比如消息之间的依赖性等等问题都需要结合具体业务场景来具体处理。

# 总结

本文主要讲述了 `RabbitMQ` 的消息发送方式，介绍了 `3` 种不同交换机的方式，同时最后也从发送消息的主要 `4` 个步骤分析了每一个步骤如何保证消息的可靠性，并分别通过 `Java API` 和 `Spring Boot` 提供了示例，中间还提到了死信队列，死信队列本质也是一个队列，只不过存储的消息比较特殊，相信通过本文，大家对 `RabbitMQ` 会有一个更深层次的了解。



