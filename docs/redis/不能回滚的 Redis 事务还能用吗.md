# 前言
经过前面相关文章的介绍，我们已经了解了Redis当中数据类型的底层存储结构，对于Redis也可以算是有了基本的认识，那么从这一篇开始，后面的文章会相继介绍Redis当中的一些高级特性，比如事务，持久化，发布/订阅/事件等等，本文主要会介绍以下两大特性：
1、Redis当中的事务及其ACID特性
2、Redis的两种持久化机制：RDB和AOF

# 事务
提到事务，我想大多数人的第一感觉就是这是关系型数据库的特性，NoSQL数据库一般都不具有事务，那么Redis作为一款NoSQL数据库有事务吗？
## Redis 有事务吗
答案是肯定的。Redis当中的单个命令都是原子操作，但是如果我们需要把多个命令组合操作的时候就需要用到事务。

Redis当中，通过下面4个命令来实现事务：
 - 1、`multi`：开启事务
 - 2、`exec`：执行事务
 - 3、`discard`：取消事务
 - 4、`watch`：监视

下图就是一个完整的事务执行流程：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20201107111844899.png#pic_center)
从上图中，我们可以总结出Redis的事务主要分为以下3步：
- 1、执行命令`multi`开启一个事务
- 2、开启事务之后执行的命令都会被放入一个队列，并且固定返回"QUEUED"
- 3、执行命令`exec`提交事务之后，会依次执行队列里面的命令，并依次返回所有命令结果(如果想要放弃事务，可以执行`discard`命令)。
## Redis事务实现原理
Redis中每个客户端都有自己的事务状态`multiState `,下面就是一个客户端`client`的数据结构定义：
```c
typedef struct client {
    uint64_t id;//客户端唯一id
    multiState mstate; //MULTI和EXEC状态(即事务状态)
    //...省略其他属性
} client;
```
`multiState `数据结构定义如下：
```c
typedef struct multiState {
    multiCmd *commands;//存储命令的FIFO队列
    int count;//命令总数
    //...省略了其他属性
} multiState;
```
`multiCmd`是一个队列用来接收并存储开启事务之后发送的命令，其数据结构定义如下：
```c
typedef struct multiCmd {
    robj **argv;//用来存储参数的数组
    int argc;//参数的数量
    struct redisCommand *cmd;//命令指针
} multiCmd;
```
我们以上面事务的示例截图中事务为例，可以得到如下所示的一个简图：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20201107161054416.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3p3eDkwMDEwMg==,size_16,color_FFFFFF,t_70#pic_center)
## Redis事务ACID特性
传统的关系型数据库中，一个事务一般都具有ACID特性，想要详细了解事务特性的可以[**点击这里**](https://blog.csdn.net/zwx900102/article/details/106544843)。那么现在就让我们来分析一下Redis是否也满足这ACID四大特性。
### A-原子性
在讨论原子性之前，我们先来看2个例子：
例子一：执行事务前报错(执行`exec`命令前)：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20201107162107378.png#pic_center)
例子二：执行事务的时候报错(执行`exec`命令时)：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20201107161803776.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3p3eDkwMDEwMg==,size_16,color_FFFFFF,t_70#pic_center)
通过上面两个例子我们发现，如果我们开启事务之后，命令在进入队列之间就报错了，那么事务将会被取消，而一旦命令成功进入队列之后，单个命令的报错就不会影响其他命令的执行，也就是**说Redis中的事务并不会回滚**。
#### Redis中的事务为什么不会滚
Redis官网中对这个问题给出了明确的解释：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20201107175409372.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3p3eDkwMDEwMg==,size_16,color_FFFFFF,t_70#pic_center)
总结起来主要就是3个原因：
- 1、Redis作者认为发生事务回滚的原因大部分都是程序错误导致，这种情况一般发生在开发阶段，而生产环境很少出现。
- 2、对于逻辑性错误，比如本来应该把一个数加1，但是程序逻辑写成了加2，那么这种错误也是无法通过事务回滚来进行解决。
- 3、Redis追求的是简单高效，而传统事务的实现相对比较复杂，这和Redis的设计思想相违背。

### C-一致性
一致性指的就是事务执行前后的数据符合数据库的定义和要求。这一点Redis是符合要求的，上面讲述原子性的时候已经提到，不论是发生语法错误还是运行时错误，错误的命令均不会被执行。
### I-隔离性
事务中的所有命令都会按顺序执行，在执行Redis事务的过程中，另一个客户端发出的请求不可能被服务，这保证了命令是作为单独的独立操作执行的。所以Redis当中的事务是符合隔离性要求的。
### D-持久性
如果Redis当中没有被开启持久化，那么就是纯内存运行的，一旦重启，所有数据都会丢失，所以不具备持久性，而如果Redis开启了持久化，那么也需要看开启的持久化模式是RDB还是AOF，还要视具体配置具体分析，这一点我们后面讲述持久化的时候会专门分析。
## WATCH命令
上面我们讲述事务的时候还提到了一个WATCH命令，这个又是做什么用的呢？我们还是先来看一个例子。
首先打开客户端1，并开启事务：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20201107181420259.png#pic_center)
上面事务中，如果这时候去执行`exec`，那么正常是第一句话返回`nil`，第二句话`ok`，第三句话`lonely_wolf`。
但是这个时候我在另一个客户端2执行一个`set name zhangsan`命令：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20201107181552421.png#pic_center)
执行成功，这时候再返回到客户端1执行`exec`命令：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20201107181626770.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3p3eDkwMDEwMg==,size_16,color_FFFFFF,t_70#pic_center)
可以发现，第一句话返回了`zhangsan`，也就是说，name这个key值在入队之后到`exec`之前发生了变化，这种在有些场景可能会导致数据被覆盖等问题的发生，那么如何解决呢？这时候`watch`命令就可以闪亮登场了。

### watch命令的作用
`watch`命令可以为Redis事务提供CAS乐观锁行为，它可以在`exec`命令执行之前，监视任意key值的变化，也就是说当多个线程更新同一个key值的时候，会跟原值做比较，一旦发现它被修改过，则拒绝执行命令，并且会返回nil给客户端。
下面还是通过一个示例来演示一下：
首先客户端1监视key值name，然后开启事务：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20201107182345532.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3p3eDkwMDEwMg==,size_16,color_FFFFFF,t_70#pic_center)
客户端2执行`set name zhangsan`命令：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20201107182439612.png#pic_center)
这时候客户端1再提交事务,会发现，事务中所有的命令都没有被执行（也就是说，只要检测到一个key值被修改过，那么整个事务都不会被执行）：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20201107182512139.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3p3eDkwMDEwMg==,size_16,color_FFFFFF,t_70#pic_center)
### watch原理分析
下面是一个Redis服务的数据结构定义：
```c
typedef struct redisDb {
    dict *watched_keys;         /* WATCHED keys for MULTI/EXEC CAS */
    int id;                     /* Database ID */
    //省略了其他属性
} redisDb;
```
可以看到，redisDb中的`watched_keys`存储了一个字典，这个字典当中的key存的就是被监视的key，然后字典的值存的就是客户端id
然后每个客户端还有一个标记属性`CLIENT_DIRTY_CAS`：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20201107204907665.png#pic_center)
一旦我们执行了一些如`set`，`sadd`等能修改key值对应value的命令，那么`CLIENT_DIRTY_CAS`标记将会被修改，后面执行事务提交命令`exec`时一旦发现这个标记被修改过，则会拒绝执行事务。