# 智能分析业务开发（六）

## 1、分析当前系统的不足

> 系统不足：只是在单机系统上进行操作

通过第五期的直播，已经经过了同步到异步的改造，但是还是存在问题。

异步实现现状：目前的异步是通过本地的线程池实现的。

### 1.1 无法集中限制，只能单机限制

假如 AI 服务限制只有2个用户同时使用，单个线程池可以限制最大核心线程数为2来实现。

假设系统用量增大，改为分布式，多台服务器，每个服务器都要有2个线程，就有可能有2N个线程，超过了AI 服务的限制。

- 解决方案：在一个集中的地方去管理下发任务（比如集中存储当前正在执行的任务数）

### 1.2 任务由于是放在内存中执行的，可能会丢失

虽然可以人工从数据库捞出来再重试，但是其实需要额外开发（比如定时任务），这种重试的场景是非常典型的，其实是不需要我们开发者过于关心、或者自己实现的。

- 解决方案：把任务放在一个可以持久化存储的硬盘

### 1.3 优化

如果你的系统功越来越多，长耗时任务越来越多，系统会越来越复杂（比如要开多个线程池、资源可能会出现项目抢占)。

- 服务拆分（应用解耦）：其实我们可以把长耗时、消耗很多的任务把它单独抽成一个程序，不要影响主业务。
- 解决方案：可以有一个中间人，让中间人帮我们去连接两个系统（比如核心系统和智能生成业务）



## 2、分布式消息队列

### 2.1 中间件

> 连接多个系统，帮助多个系统紧密协作的技术（或者组件）。

比如：Redis、消息队列、分布式存储Etcd

![image-20230618161215010](https://paperfly-blog.oss-cn-shenzhen.aliyuncs.com/blog/image-20230618161215010.png)

### 2.2 消息队列

> 消息队列：存储消息的队列
>
> 
>
> 消息队列：一种在应用程序之间传递消息的通信机制。它类似于实际世界中的一个邮箱系统，发送者将消息放入消息队列，接收者从队列中获取消息进行处理。这种异步的方式可以提高系统的性能和可靠性，因为发送者和接收者可以独立地进行操作，并且不需要直接相互依赖。
>
> 
>
> 消息队列允许不同的应用程序、服务或组件之间解耦，它们可以独立地进行伸缩和维护。消息队列还提供了**可靠性和持久性**的保证，因为消息可以在发送时进行持久化，并且在接收者处理之前可以进行多次尝试。



消息队列包含的关键词：存储、消息、队列

- 存储：存储数据
- 消息：某种数据结构：比如字符串、对象、二进制、JSON等等
- 队列：一种**先进先出的数据结构**

#### 应用场景（作用）

在多个不同的系统、应用之间实现消息的传输（也可以存储）。不需要考虑传输应用的编程语言、系统、框架等等。

> 例如：可以让Java开发的应用发消息，让PHP开发的应用收消息，这样就不用把所有代码写到同一个项目里（应用解耦)



### 2.3 消息队列模型

- 生产者：Producer，类比为快递员，发送消息的人（客户端）

- 消费者：Consumer，类比为取快递的人，接受读取消息的人（客户瑞）

- 消息：Message，类比为你买的快递，就是生产者要传输给消费者的数据

- 消息队列：Queue：类比为快递接受柜


![image-20230618163709919](https://paperfly-blog.oss-cn-shenzhen.aliyuncs.com/blog/image-20230618163709919.png)

为什么不接传输，要用消息队列？

> 生产者不用关心消费者要不要消费、什么时候消费，生产者只需要把东西（消息）给消息队列，生产者的工作就算完成了。
>
> 
>
> 比如上面这个送快递的例子：
>
> 送快递的配送员，只需要将快递放到快递柜，发个消息给你，快递已经放到了快递柜，那么你就可以在有空的时候再去拿这个快递，而快递配送员也不用管你什么时候去拿（不弄丢就可以，弄丢那就得另外说了）

生产者和消费者实现了解耦，互不影响。达到了**不需要考虑传输应用的编程语言、系统、框架**等等。





### 2.4 为什么要用消息队列

使用消息队列有以下几个主要原因：

1. 解耦和提高系统可靠性：消息队列提供了解耦的方式，发送方和接收方之间不直接进行通信，通过消息队列中转。这样，即使其中一个组件或者系统出现故障，不会影响到整个系统的正常运行。

2. 异步处理和提高系统性能：消息队列可以使发送方和接收方异步处理消息。发送方将消息发送到队列中后，不需要等待接收方立即处理，而是可以继续处理其他任务。这样可以提高系统的性能和响应速度。

3. 缓冲和流量控制：消息队列可以作为缓冲区，用于处理流量峰值。当系统面临突发的请求量增加时，消息队列可以缓冲请求并逐渐处理，避免系统过载。

4. 同步数据：在分布式系统中，不同组件或者系统之间可能需要共享数据。通过消息队列可以实现数据的同步和共享，保证各个组件之间的数据一致性。

5. 扩展和灵活性：使用消息队列可以通过增加消费者来实现系统的扩展性，并且消费者可以根据需求灵活地进行扩缩容。这样可以根据实际需求进行系统的调整和优化。



#### 2.4.1 异步处理

生产者发送完消息之后，可以继续去忙别的，消费者想什么时候消费都可以，不会产生阻塞。

#### 2.4.2 削峰填谷

先把用户的请求放到消息队列中，消费者（实际执行操作的应用）可以按照自己的需求，慢慢去取。

**没有使用消息队列**：12点时来了10万个请求，原本情况下，10万个请求都在系统内部立刻处理，很快系统压力过大就宕机了。

**使用消息队列**：把这10万个请求放到消息队列中，处理系统以自己的恒定速率（比如每秒1个）慢慢执行，从而保护系统、稳定处理。



### 2.5 分布式消息队列的优势

1. 数据持久化：它可以把消息集中存储到硬盘里，服务器重启就不会丢失
2. 可扩展性：可以根据需求，随时增加（或减少）节点，继续保持稳定的服务
3. 应用解耦：可以连接各个不同语言、框架开发的系统，上这些系统能够灵活传输读取数据
4. 发布订阅：它可以使发布者**将消息发送到一个或多个订阅者**，从而实现解耦、可伸缩性和实时性等优势
   1. 解耦性：发布者和订阅者之间是松耦合的，彼此不需要直接通信。发布者只需要将消息发布到特定的主题或频道中，而订阅者只需要订阅感兴趣的主题或频道。这种解耦性使得系统中的组件可以独立地进行开发、维护和扩展。
   2. 可伸缩性：发布订阅模式允许多个订阅者同时接收消息，并且可以动态地添加或移除订阅者。这种可伸缩性使系统能够处理更高的并发量，而不会对性能和可用性产生负面影响。
   3. 实时性：由于发布订阅模式支持多个订阅者同时接收消息，可以更快地将消息传递给订阅者。这种实时性使系统能够实现即时的数据传输和处理，适用于需要快速响应的场景。
   4. 可靠性：分布式消息队列通常提供了消息持久化的机制，可以将消息存储到磁盘上，确保消息不会丢失。即使消费者出现故障，稍后再启动时也可以继续接收之前未处理的消息。

#### 2.5.1 应用解耦优点

不使用消息队列时：

> 把所有的功能都放在同一个项目中，调用多个子功能时，一个环节错，整个系统就出错

![image-20230618164212956](https://paperfly-blog.oss-cn-shenzhen.aliyuncs.com/blog/image-20230618164212956.png)



使用了消息队列进行解耦：

1. 一个系统挂了，不影响另一个系统：如果发货系统挂了，但是不影响到库存系统进行扣减
2. 系统挂了并恢复后，仍然可以从消息队列当中取出消息，继续执行业务逻辑
3. 只要发送消息到队列，就可以立刻返回，不用同步调用所有系统，性能更高

![image-20230618164518027](https://paperfly-blog.oss-cn-shenzhen.aliyuncs.com/blog/image-20230618164518027.png)

> 订单系统只要将消息发送到消息队列，即可返回，并不需要调用其他的系统，而其他的系统只要从消息队列中获取到消息。



发布订阅优势图解：

如果一个非常大的系统要给其他子系统发送通知，最简单直接的方式是**大系统直接依次调用小系统**

> 比如QQ：使用QQ来关联了很多应用的消息，比如王者、吃鸡、微信等等

![image-20230618165359369](https://paperfly-blog.oss-cn-shenzhen.aliyuncs.com/blog/image-20230618165359369.png)

这样做存在的问题：

1. 每次发通知都要调用很多系统，很麻烦、有可能失败
2. 新出现的项目（或者说大项目感知不到的项目）无法得到通知
3. 发布的消息不知道哪个系统需要

解决方案：大的核心系统始终往一个地方（消息队列）去发消息，其他的系统都去订阅这个消息队列（读取这个消息队列中的消息)

![image-20230618165720318](https://paperfly-blog.oss-cn-shenzhen.aliyuncs.com/blog/image-20230618165720318.png)



### 2.6 分布式消息队列的缺点

1. 增加复杂性：分布式消息队列的部署和运维需要一定的技术知识和经验，对系统架构和设计有一定的要求。同时，因为引入了额外的组件，**系统的复杂度也会增加**，系统会更复杂、额外维护中间件、额外的费用（部署）成本
2. 消息队列：消息丢失、顺序性、重复消费、数据的一致性（分布式系统就要考虑一致性问题）
3. 消息顺序性：由于消息队列的异步性，无法保证消息的顺序性。这可能会对依赖消息顺序的业务场景造成影响，需要额外的处理机制来解决此问题。







### 2.7 分布式消息队列应用场景

1. 耗时的场景（异步）
2. 分布式系统协作（尤其是跨团队、跨业务协作，应用解耦）
3. 强稳定性的场景（比如金融业务：支付、转账，特久化、可靠性、削峰填谷）
4. 异步通信：当系统需要在**不同模块之间进行异步通信**时，分布式消息队列可以提供一种可靠的通信机制。
5. 解耦应用：当**多个应用或服务之间需要解耦**，以降低耦合度、提高灵活性和可维护性时，使用分布式消息队列可以实现解耦。
6. 流量削峰：在**高峰期或服务器负载高的情况下**，分布式消息队列可以将**流量平滑分发到不同的消费者**上，避免系统过载。（**高并发场景、异步、削峰填谷**）
7. 日志处理：对于**大规模的日志处理场景**，分布式消息队列可以帮助将日志收集、传输和处理进行解耦，并且可以容错和持久化。
8. 事件驱动架构：在事件驱动的架构中，各个组件通过订阅消息实现解耦，以便在事件发生时进行相应的处理。



### 2.8 主流分布式消息队列选型

以下是对主流分布式消息队列选型的简要介绍：

1. ActiveMQ：是一个开源的、完全支持JMS规范的分布式消息队列系统。它提供了可靠的消息传递、事务、消息持久化等功能，适用于大多数企业级应用场景。
2. RabbitMQ：也是一个开源的分布式消息队列系统，它实现了AMQP协议。RabbitMQ具有高可用性、可靠性和灵活的路由策略，适用于复杂的消息路由和灵活的消息处理场景。
3. Kafka：是一个分布式流处理平台和消息队列系统，提供高吞吐量、低延迟的消息传递。Kafka适用于大数据流处理场景，例如日志收集、实时分析等。
4. RocketMQ：是一个开源的分布式消息队列系统，最初由阿里巴巴开发，现已进入Apache孵化器。RocketMQ具有高性能、高可用性和分布式事务等特性，适用于大规模的分布式应用场景。
5. ZeroMQ：是一个轻量级的消息队列库，支持多种通信模式和传输协议。ZeroMQ的设计目标是提供简单的消息传递机制，适用于轻量级的异步通信和快速构建分布式系统的场景。
6. Pulsar：是一个开源的分布式消息和流处理平台，由Apache孵化器支持。Pulsar提供了高可伸缩性、多租户、低延迟等特性，并支持云原生架构，适用于大规模的数据处理和实时分析场景。
7. Apache InLong (Tube)：前身为Apache Tube，是一个开源的分布式消息和流式数据传输平台。它提供了灵活的消息传递和数据传输方式，并集成了大数据处理能力，适用于实时分析、数据集成等多样化场景。



#### 2.8.1 不同消息队列对比

主要是参考以下几个技术指标进行选择对应的消息队列来开发

1. 吞吐量：IO、并发。（表示单位时间内完成的操作次数、传输的数据量或处理的任务数量）

2. 时效性：类以延迟，消息的发送、到达时间

3. 可用性：系统可用的比率（比如1年365天宕机1s，可用率大概X个9）

   > 来自鱼聪明：
   >
   > 可用性的计算公式是：可用时间 / (总时间 - 计划停机时间)。
   >
   > 总时间 = 365 * 24 * 60 * 60 = 31,536,000 秒。
   >
   > 计划停机时间 = 1 秒 * 365 天 = 365 秒。
   >
   > 可用时间 = 总时间 - 计划停机时间 = 31,536,000 - 365 = 31,535,635 秒。
   >
   > 可用性 = 31,535,635 / 31,536,000 ≈ 0.999988 ≈ 99.9988%。
   >
   > 所以，该系统的可用性为约99.9988%。

4. 可靠性：消息不丢失（比如不丢失订单）、功能正常完成

> - 吞吐量（Throughput）指的是系统在**单位时间内能够处理的请求或传输的数据量**。在消息队列中，吞吐量表示系统能够处理的消息数量或传输的数据量。较高的吞吐量意味着系统能够更快地处理请求或传输数据。
> - 时效性（Latency）指的是**从发出请求到收到响应所经历的时间间隔**。在消息队列中，时效性表示从消息被发送到消息被接收和处理的时间。较低的时效性意味着消息能够更快地被传输、接收和处理。

| 消息队列             | 吞吐量（QPS）  | 时效性           | 可用性 | 可靠性 | 优势                                                         | 缺点                         | 应用场景                                                     |
| -------------------- | -------------- | ---------------- | ------ | ------ | ------------------------------------------------------------ | ---------------------------- | ------------------------------------------------------------ |
| ActiveMQ             | 中等：万级     | 高               | 高     | 中等   | 成熟的JMS支持、简单易学                                      | 性能相对较低                 | 中小型企业应用，可靠的消息传递和事务                         |
| RabbitMQ             | 中等：万级     | 极高（微秒单位） | 高     | 高     | 灵活的路由策略、生态好、时效性高、易学                       | 对大吞吐量的支持相对较弱     | 复杂的消息路由和灵活的消息处理、适用于大部分的分布式系统     |
| Kafka                | **高：十万级** | 高（毫秒以内）   | 极高   | 极高   | 高吞吐量和低延迟、可靠性、强大的数据流处理能力               | 复杂性较高，初学者上手难度大 | 大数据流处理，日志收集，实时数据流传输、事件流收集传输等     |
| RocketMQ             | 高：十万级     | 高（毫秒）       | 极高   | 极高   | 可靠性、可用性、吞吐量大、分布式事务支持                     | 对复杂性的支持相对不足       | 大规模的分布式应用，高可用性和事务一致性要求<br />适用于**金融**、电商等对可靠性要求较高的场景，适合大规模的消息处理 |
| ZeroMQ               | 中等：万级     | 高               | 高     | 低     | 简单的消息传递机制                                           | 缺乏对复杂消息路由的支持     | 轻量级异步通信，构建快速分布式系统                           |
| Pulsar               | 高：十万级     | 高（毫秒）       | 极高   | 极高   | 可靠性、可用性很高、基于发布订阅模型、新兴技术结构、云原生架构支持 | 部署和运维相对复杂           | 适用大规模数据处理，实时分析，高并发的分布式系统。适合实时分析、事件流处理、IoT数据处理等。 |
| Apache InLong (Tube) | 高：十万级     | 高               | 高     | 高     | 数据传输和大数据处理集成                                     | 社区生态相对较小             | 实时分析，数据集成                                           |

 

## 3、RabbitMQ 入门实践

[空降官网](https://www.rabbitmq.com/)

特点：生态好，好学习、易于理解，时效性强，支特很多不同语言的客户端，扩展性、可用性都很不错。

学习性价比非常高的消息队列，适用于绝大多数中小规模分布式系统。

 

### 3.1 基本消息

AMQP协议（高消息队列协议：Advanced Message Queuing Protocol）：https://www.rabbitmq.com/tutorials/amqp-concepts.html

![](https://paperfly-blog.oss-cn-shenzhen.aliyuncs.com/blog/hello-world-example-routing.png)

- 生产者(Publisher)：发消息到某个交换机
- 消费者(Consumer)：从某个队列中取消息
- 交换机(Exchange)：负责把消息转发到对应的队列
- 队列(Queue)：存储消息的
- 路由(Routes)：转发，就是怎么把消息从一个地方转到另一个地方（比如从生产者转发到某个队列）





### 3.2 安装 RabbitMQ 

#### 3.2.1 Windows安装

Windows官网安装教程：https://www.rabbitmq.com/install-windows.html



安装Erlang 25.3.2 ：因为Rabbit MQ 依赖于erlang ，它的性能非常高

Erlang 下载：https://www.erlang.org/patches/otp-25.3.2



Windows 安装 Rabbit MQ 监控面板，执行下面的命令：

```
rabbitmq-plugins.bat enable rabbitmq_management
```

Rabbit MQ 端口：

1. 5672：程序链接的端口
2. 15672：WebUI

#### 3.2.2 Linux 安装

安装Docker

```
sudo yum install docker-ce
```

安装完Docker后，启动Docker服务：

```
sudo systemctl start docker
```

通过 docke r拉取 rabbitmq 镜像：

```
sudo docker pull rabbitmq:版本号
docker pull rabbitmq:3.9-management
```

请将 "版本号" 替换为您想要的RabbitMQ版本号。如果不指定版本号，则会拉取最新的稳定版本。

完成镜像拉取后，可以使用以下命令启动RabbitMQ容器：

```
sudo docker run -d --name myrabbitmq -p 5672:5672 -p 15672:15672 rabbitmq:版本号
sudo docker run -d --name myrabbitmq -p 5672:5672 -p 15672:15672 rabbitmq:3.9-management
```

同样，请将"版本号"替换为您之前选择的RabbitMQ版本号。

容器启动后，可以使用以下命令检查容器是否正在运行：

```
sudo docker ps
```

如果能看到刚刚启动的RabbitMQ容器正在运行，那么恭喜您已成功在CentOS上使用Docker安装了RabbitMQ。

还在将服务器的1562、5672端口开放，

最后，通过浏览器访问 `http:// 服务器IP地址:15672` 来打开RabbitMQ的WebUI管理界面，默认的用户名和密码是"guest"。可以进行用户设置、队列管理等操作。



Linux下的RabbitMQ新增一个用户，分别执行以下的命令：

```bash
# 查看rabbitmq 运行的容器id
dcker ps
## 进入正在运行的 RabbitMQ 容器的 Shell。执行以下命令（将 <container_id> 替换为实际的容器 ID）：
docker exec -it <container_id> /bin/bash
# 添加新用户
rabbitmqctl add_user 用户名 密码
# 管理员权限
rabbitmqctl set_user_tags 上面的用户名 administrator 
# 列出用户列表
rabbitmqctl list_users
# 目录权限
# 第一个".*" 用于在每个实体上配置权限
# 第二个".*" 表示对每个实体的写入权限
# 第三个".*” 对每个实体的读取权限
rabbitmqctl set_permissions -p "/" "用户名" ".*" ".*" ".*"
```



### 3.3 快速入门

#### 3.3.1 单向发送

> 一个生产者给一个队列发送消息，一个消费者从这个队列中获取到消息。1 对 1 关系

![image-20230617172656480](https://paperfly-blog.oss-cn-shenzhen.aliyuncs.com/blog/image-20230617172656480.png)

引入Rabbit MQ 组件依赖

```xml
<!-- https://mvnrepository.com/artifact/com.rabbitmq/amqp-client -->
<dependency>
    <groupId>com.rabbitmq</groupId>
    <artifactId>amqp-client</artifactId>
    <version>5.16.0</version>
</dependency>
```



生产者代码：

```java
/**
 * @author Shier
 */
public class SingleProducer {
    private final static String SINGLE_QUEUE_NAME = "hello";

    public static void main(String[] argv) throws Exception {
        // 创建链接工厂
        ConnectionFactory factory = new ConnectionFactory();
        // 设置 rabbitmq 对应的信息
        factory.setHost("xx.xx.xx.xx");
        factory.setUsername("xxx");
        factory.setPassword("xxx");
        try (Connection connection = factory.newConnection();
             // 建立链接，创建频道
             Channel channel = connection.createChannel()) {
            // 创建消息队列
            channel.queueDeclare(SINGLE_QUEUE_NAME, false, false, false, null);
            String message = "Hello World!";
            channel.basicPublish("", SINGLE_QUEUE_NAME, null, message.getBytes(StandardCharsets.UTF_8));
            System.out.println(" [x] Sent '" + message + "'");
        }
    }
}
```



Channel 频道：理解为操作消息队列的client(比如 JDBCClient、redisClient)，提供了和消息队列server建立通信的传输方法（为了复用连接，提高传输效率）。程序通过channel操作 rabbitmq (收发消息)

```java
Queue.DeclareOk queueDeclare(String queue, boolean durable, boolean exclusive, boolean autoDelete,
                             Map<String, Object> arguments) throws IOException;
```

创建消息队列的参数说明：

- queueName：消息队列名称（注意，同名称的消息队列，只能用同样的参数创建一次）
- durabale：消息队列重启后，消息是否丢失，持久化处理
- exclusive：是否**只允许当前这个创建消息队列**的连接操作消息队列

- autoDelete：没有人用队列后，是否要删除队列

- arguments：用于设置队列的其他属性。可以通过传递一个键值对的Map来指定这些属性。




消费者代码：

```java
/**
 * @author Shier
 */
public class SingleConsumer {

    private final static String QUEUE_NAME = "hello";

    public static void main(String[] argv) throws Exception {
        // 建立链接，
        ConnectionFactory factory = new ConnectionFactory();
        // 设置 rabbitmq 对应的信息
        factory.setHost("xx.xx.xx.xx");
        factory.setUsername("xxx");
        factory.setPassword("xxx");

        Connection connection = factory.newConnection();
        Channel channel = connection.createChannel();

        // 创建队列
        channel.queueDeclare(QUEUE_NAME, false, false, false, null);
        System.out.println(" [*] Waiting for messages. To exit press CTRL+C");

        // 定义如何处理消息
        DeliverCallback deliverCallback = (consumerTag, delivery) -> {
            String message = new String(delivery.getBody(), StandardCharsets.UTF_8);
            System.out.println(" [x] Received '" + message + "'");
        };
        // 消费消息
        channel.basicConsume(QUEUE_NAME, true, deliverCallback, consumerTag -> { });
    }
}
```





#### 3.3.2 多消费者 （使用较少）

> 场景：多个机器同时去接受并处理任务（尤其是每个机器的处理能力有限）
>
> 一个生产者给一个队列发消息，多个消费者从这个队列取消息。1 对 多关系。

![image-20230617173154369](https://paperfly-blog.oss-cn-shenzhen.aliyuncs.com/blog/image-20230617173154369.png)

1）队列持久化

第二个参数：durable 参数设置为 true ，服务器重启后队列不丢失

```java
channel.queueDeclare(TASK_QUEUE_NAME, true, false, false, null);
```

2）消息持久化

将 basicPublish 中的第二个参数指定为 MessageProperties.PERSISTENT_TEXT_PLAIN ：

```java
channel.basicPublish("", TASK_QUEUE_NAME,
       MessageProperties.PERSISTENT_TEXT_PLAIN,
       message.getBytes("UTF-8"));
```

- ""：表示交换机的名称，这里为空字符串，表示使用默认的交换机。
- TASK_QUEUE_NAME：表示要发布消息的队列的名称。
- MessageProperties.PERSISTENT_TEXT_PLAIN：指定消息的属性，这里是设置消息持久化。PERSISTENT_TEXT_PLAIN是一个MessageProperties对象，它可以设置消息的各种属性，比如持久化、优先级等。
- message.getBytes("UTF-8")：将消息内容转换为字节数组，并指定字符集为UTF-8。

这个方法会将消息发送到指定的队列中，如果队列不存在则会抛出异常。消息的持久化属性设置为true，表示消息会被存储到磁盘上，即使在服务器重启后也能保留。



3）生产者代码：

使用 Scanner 接受用户输入，便于发送多条消息

```java
/**
 * @author Shier
 */
public class MultiProducer {

    private static final String TASK_QUEUE_NAME = "multi_queue";

    public static void main(String[] argv) throws Exception {

        ConnectionFactory factory = new ConnectionFactory();
        // 设置 rabbitmq 对应的信息
        factory.setHost("xxxxx.xxxx.xxxx");
        factory.setUsername("xxxx.xxxx.xxx");
        factory.setPassword("xxx.xxx.xxx");
        // 建立链接
        try (Connection connection = factory.newConnection();
             Channel channel = connection.createChannel()) {
            // 创建队列
            channel.queueDeclare(TASK_QUEUE_NAME, true, false, false, null);

            Scanner scanner = new Scanner(System.in);
            while (scanner.hasNext()){
                String message = scanner.nextLine();
                channel.basicPublish("", TASK_QUEUE_NAME,
                        MessageProperties.PERSISTENT_TEXT_PLAIN,
                        message.getBytes("UTF-8"));
                System.out.println(" [x] Sent '" + message + "'");
            }
        }
    }
}
```

消费者代码：

```java
/**
 * @author Shier
 */
public class MultiConsumer {

    private static final String TASK_QUEUE_NAME = "multi_queue";

    public static void main(String[] argv) throws Exception {
        // 建立链接
        ConnectionFactory factory = new ConnectionFactory();
        // 设置 rabbitmq 对应的信息
        factory.setHost("xxxxx.xxxx.xxxx");
        factory.setUsername("xxxx.xxxx.xxx");
        factory.setPassword("xxx.xxx.xxx");
        final Connection connection = factory.newConnection();

        for (int i = 0; i < 2; i++) {
            // 创建两个消费者频道
            final Channel channel = connection.createChannel();
            channel.queueDeclare(TASK_QUEUE_NAME, true, false, false, null);
            System.out.println(" [*] Waiting for messages. To exit press CTRL+C");

            // 控制当个消费者的积压数
            channel.basicQos(1);

            int finalI = i;
            DeliverCallback deliverCallback = (consumerTag, delivery) -> {

                String message = new String(delivery.getBody(), "UTF-8");
                try {
                    // 处理工作
                    System.out.println(" [x] Received '消费者：" + finalI + "，消费了：" + message + "'");
                    // 指定取某条消息
                    channel.basicAck(delivery.getEnvelope().getDeliveryTag(),false);
                    // 停20秒 模拟机器处理能力有限
                    Thread.sleep(10000);
                } catch (InterruptedException e) {
                    e.getStackTrace();
                    // 第三个参数
                    channel.basicNack(delivery.getEnvelope().getDeliveryTag(),false,false);
                } finally {
                    System.out.println(" [x] Done");
                    channel.basicAck(delivery.getEnvelope().getDeliveryTag(), false);
                }
            };
            // 开启消费监听，会一直监听生产者的消息
            channel.basicConsume(TASK_QUEUE_NAME, false, deliverCallback, consumerTag -> {
            });
        }
    }
}
```

控制单个消费者的处理任务积压数：每个消费者最多同时处理1个任务

```java
channel.basicQos(1);
```



##### **消息确认机制** （面试爱问）

```java
channel.basicConsume(TASK_QUEUE_NAME, false, deliverCallback, consumerTag -> {});
// basicConsume()方法源码：
String basicConsume(String queue, boolean autoAck, DeliverCallback deliverCallback, CancelCallback cancelCallback) throws IOException;
```

为了保证消息成功被消费（快递成功被取走），rabbitmg提供了消息确认机制，当消费者接收到消息后，比如要给一个反馈：

- ack：消费成功
- nack：消费失败
- reject：拒绝

> 如果告诉 Rabbit MQ 服务器消费成功，服务器才会放心地移除消息。
>
> 支持配置autoack，会自动执行 ack 命令，接收到消息立刻就成功了。
>
> 建议 autoack 改为 false ，根据实际情况，去手动确认。



**指定确认某条消息**：

```java
channel.basicAck(delivery.getEnvelope().getDeliveryTag(),false);
// 源码
void basicAck(long deliveryTag, boolean multiple) throws IOException;
```

> 参数解释：
>
> 1. `deliveryTag`：表示要确认的消息的标识符。每个消息都有一个唯一的`deliveryTag`，用于标识消息的顺序。
> 2. `multiple`：表示是否批量确认消息。如果设置为`true`，则表示确认所有在`deliveryTag`之前的未确认消息；如果设置为`false`，则只确认当前`deliveryTag`的消息。
>
> 第二参数 multiple 批量确认：是指是否要一次性确认所有的历史消息直到当前这条



**指定拒绝某条消息**：

```java
channel.basicNack(delivery.getEnvelope().getDeliveryTag(),false,false);
// 源码
void basicNack(long deliveryTag, boolean multiple, boolean requeue)
            throws IOException;
```

> 参数解释：
>
> 1. `deliveryTag`：表示要否定确认的消息的标识符。每个消息都有一个唯一的`deliveryTag`，用于标识消息的顺序。
> 2. `multiple`：表示是否批量否定确认消息。如果设置为`true`，则表示否定所有在`deliveryTag`之前的未确认消息；如果设置为`false`，则只否定当前`deliveryTag`的消息。
> 3. `requeue`：表示是否将消息重新放回队列。如果设置为`true`，则消息将被重新放回队列并可以被其他消费者重新消费；如果设置为`false`，则消息将会被丢弃。
>
> 第三个参数表示是否重新入队，可用于重试



**2个测试小技巧：**

1. 使用Scanner接受用户输入，便于快速发送多条消息
2. 使用 for 循环创建多个消费者，便于快速验证队列模型工作机制



### 3.4 Rabbit MQ 交换机

一个生产者给多个队列发消息，1个生产者对多个队列。

交换机的作用：提供消息转发功能，类以于网络路由器

要解决的问题：怎么把消息转发到不同的队列上，好让消费者从不同的队列消费。



绑定：交换机和队列关联起来，也可以叫路由，算是一个算法或转发策略

绑定代码方法：

```java
channel.exchangeDeclare(FANOUT_EXCHANGE_NAME, "fanout");
```

![](https://cdn.nlark.com/yuque/0/2023/png/398476/1686839285368-841c3ded-965b-4ed7-8214-091ac7f2c922.png)

> 后面的就是不同的队列



#### 3.4.1 交换机类型

1. **Fanout Exchange**（广播类型）：Fanout交换机**将消息广播到其绑定的所有队列**。当消息被发送到Fanout交换机时，它会将消息复制到所有绑定的队列上，而不考虑路由键的值。因此，无论消息的路由键是什么，都会被广播到所有队列。Fanout交换机主要用于广播消息给所有的消费者。
2. **Direct Exchange**（直连类型）：Direct交换机是**根据消息的路由键选择将消息路由到与消息具有相同路由键绑定的队列**。例如，当消息的路由键与绑定键完全匹配时，消息将被路由到对应的队列。Direct交换机主要用于一对一的消息路由。
3. **Topic Exchange**（主题类型）：Topic交换机**将消息根据路由键的模式进行匹配，并将消息路由到与消息的路由键匹配的队列**。路由键可以使用通配符匹配，支持两种通配符符号，"#"表示匹配一个或多个单词，"*"表示匹配一个单词。Topic交换机主要用于灵活的消息路由。
4. **Headers Exchange**（头类型）：Headers交换机是**根据消息的头部信息进行匹配，并将消息路由到匹配的队列**。头部信息通常是一组键值对，可以使用各种自定义的标准和非标准的头部信息进行匹配。Headers交换机主要用于复杂的匹配规则。



#### 3.4.2 fanout 交换机

扇出、广播消息

特点：消息会被转发到所有绑定到该交换机的队列

场景：很适用于发布订阅的场景。比如写日志，可以多个系统间共享

![](https://shierimages.oss-cn-shenzhen.aliyuncs.com/TyporaImages/1686839285368-841c3ded-965b-4ed7-8214-091ac7f2c922.png)

实例场景：

![image-20230619225118563](https://shierimages.oss-cn-shenzhen.aliyuncs.com/TyporaImages/image-20230619225118563.png)



生产者代码：

```java
/**
 * @author Shier
 */
public class FanoutProducer {
    private static final String FANOUT_EXCHANGE_NAME = "fanout-exchange";

    public static void main(String[] argv) throws Exception {
        ConnectionFactory factory = new ConnectionFactory();
        // 设置 rabbitmq 对应的信息
        factory.setHost("xxxxx.xxxx.xxxx");
        factory.setUsername("xxxx.xxxx.xxx");
        factory.setPassword("xxx.xxx.xxx");
        try (Connection connection = factory.newConnection();
             Channel channel = connection.createChannel()) {
            // 创建交换机
            channel.exchangeDeclare(FANOUT_EXCHANGE_NAME, "fanout");
            // 发送给所有的队列，所以说不要写队列名称
            Scanner scanner = new Scanner(System.in);
            while (scanner.hasNext()) {
                String message = scanner.nextLine();
                channel.basicPublish(FANOUT_EXCHANGE_NAME, "", null, message.getBytes("UTF-8"));
                System.out.println(" [x] Sent '" + message + "'");
            }

        }
    }
}
```

消费者代码：

1. 消费者和生产者要绑定同一个交换机
2. 先有队列，才能进行绑定

```java
/**
 * @author Shier
 */
public class FanoutConsumer {
    private static final String FANOUT_EXCHANGE_NAME = "fanout-exchange";

    public static void main(String[] argv) throws Exception {
        ConnectionFactory factory = new ConnectionFactory();
        // 设置 rabbitmq 对应的信息
        factory.setHost("xxxxx.xxxx.xxxx");
        factory.setUsername("xxxx.xxxx.xxx");
        factory.setPassword("xxx.xxx.xxx");

        Connection connection = factory.newConnection();

        Channel channel1 = connection.createChannel();
        Channel channel2 = connection.createChannel();

        // 声明交换机
        channel1.exchangeDeclare(FANOUT_EXCHANGE_NAME, "fanout");
        //channel2.exchangeDeclare(FANOUT_EXCHANGE_NAME, "fanout");

        // 员工小红
        String queueName1 = "xiaohong_queue";
        channel1.queueDeclare(queueName1, true, false, false, null);
        channel1.queueBind(queueName1, FANOUT_EXCHANGE_NAME, "");
        // 员工小蓝
        String queueName2 = "xiaolan_queue";
        channel2.queueDeclare(queueName2, true, false, false, null);
        channel2.queueBind(queueName2, FANOUT_EXCHANGE_NAME, "");

        System.out.println(" [*] =========================================");

        DeliverCallback deliverCallback1 = (consumerTag, delivery) -> {
            String message = new String(delivery.getBody(), "UTF-8");
            System.out.println(" [x] Received '" + message + "'");
        };

        DeliverCallback deliverCallback2 = (consumerTag, delivery) -> {
            String message = new String(delivery.getBody(), "UTF-8");
            System.out.println(" [x] Received '" + message + "'");
        };

        // 监听消息
        channel1.basicConsume(queueName1, true, deliverCallback1, consumerTag -> {
        });
        channel2.basicConsume(queueName2, true, deliverCallback2, consumerTag -> {
        });
    }
}
```

> 效果：所有的消费者都能收到消息

#### 3.4.3 Direct 交换机

官网介绍教程：https://www.rabbitmq.com/tutorials/tutorial-four-java.html

绑定：可以让交换机和队列进行关联，可以指定让交互机把什么祥的消息发送给哪个队列（类以于计算机网络中，两个路由器，或者网络设备相互连接，也可以理解为网线)

routing Key：路由键，控制消息要转发给**指定的那个队列**的 （可以简单的说是 IP地址）

特点：消息**会根据路由键转发到指定的队列**

场景：特定的消息只交给特定的系统（程序）来处理

绑定关系：完全匹配字符串，路由键要完全匹配

![image-20230620131236850](https://shierimages.oss-cn-shenzhen.aliyuncs.com/TyporaImages/image-20230620131236850.png)



不同的队列也可以绑定同样的路由键。

比如发日志的场景，希望用独立的程序来处理不同级别的日志，比如 C1 系统处理 error 日志，C2系统处理其他级别的日志

![image-20230620163511572](https://shierimages.oss-cn-shenzhen.aliyuncs.com/TyporaImages/image-20230620163511572.png)



**示例场景**

![image-20230620165023953](https://shierimages.oss-cn-shenzhen.aliyuncs.com/TyporaImages/image-20230620165023953.png)

生产者代码：

```java
/**
 * @author Shier
 */
public class DirectProducer {

    private static final String DIRECT_EXCHANGER = "direct-exchange";

    public static void main(String[] argv) throws Exception {
        ConnectionFactory factory = new ConnectionFactory();

        // 设置 rabbitmq 对应的信息
        factory.setHost("xxxxx.xxxx.xxxx");
        factory.setUsername("xxxx.xxxx.xxx");
        factory.setPassword("xxx.xxx.xxx");

        try (Connection connection = factory.newConnection();
             Channel channel = connection.createChannel()) {
            channel.exchangeDeclare(DIRECT_EXCHANGER, "direct");

            Scanner scanner = new Scanner(System.in);
            while (scanner.hasNext()) {
                String userInput = scanner.nextLine();
                String[] splits = userInput.split(" ");
                if (splits.length < 1) {
                    continue;
                }
                String message = splits[0];
                String routingKey = splits[1];

                channel.basicPublish(DIRECT_EXCHANGER, routingKey, null, message.getBytes("UTF-8"));
                System.out.println(" [x] Sent '" + message + " with routing " + routingKey + "'");
            }
        }
    }
}
```

消费者代码：

```java
/**
 * @author Shier
 */
public class DirectConsumer {

    private static final String DIRECT_EXCHANGE = "direct-exchange";

    public static void main(String[] argv) throws Exception {
        ConnectionFactory factory = new ConnectionFactory();
        // 设置 rabbitmq 对应的信息
        factory.setHost("xxxxx.xxxx.xxxx");
        factory.setUsername("xxxx.xxxx.xxx");
        factory.setPassword("xxx.xxx.xxx");

        Connection connection = factory.newConnection();
        Channel channel = connection.createChannel();

        channel.exchangeDeclare(DIRECT_EXCHANGE, "direct");

        // 创建队列，分配一个队列名称：小紫
        String queueName = "xiaozi_queue";
        channel.queueDeclare(queueName, true, false, false, null);
        channel.queueBind(queueName, DIRECT_EXCHANGE, "xiaozi");

        // 创建队列，分配一个队列名称：小黑
        String queueName2 = "xiaohei_queue";
        channel.queueDeclare(queueName2, true, false, false, null);
        channel.queueBind(queueName2, DIRECT_EXCHANGE, "xiaohei");

        System.out.println(" [*] Waiting for messages. To exit press CTRL+C");

        // 小紫队列监听机制
        DeliverCallback xiaoziDeliverCallback = (consumerTag, delivery) -> {
            String message = new String(delivery.getBody(), "UTF-8");
            System.out.println(" [xiaozi] Received '" +
                    delivery.getEnvelope().getRoutingKey() + "':'" + message + "'");
        };
        // 小黑队列监听机制
        DeliverCallback xiaoheiDeliverCallback = (consumerTag, delivery) -> {
            String message = new String(delivery.getBody(), "UTF-8");
            System.out.println(" [xiaohei] Received '" +
                    delivery.getEnvelope().getRoutingKey() + "':'" + message + "'");
        };
        channel.basicConsume(queueName, true, xiaoziDeliverCallback, consumerTag -> {
        });
        channel.basicConsume(queueName2, true, xiaoheiDeliverCallback, consumerTag -> {
        });
    }
}
```



#### 3.4.4 topic 交换机

官网教程：https://www.rabbitmq.com/tutorials/tutorial-five-java.html

特点：消息会根据一个模糊的路由键转发到指定的队列

场景：特定的一类消息可以交给特定的一类系统（程序）来处理

绑定关系：可以模湖匹配多个绑定

1. `*`：匹配一个单词，比如`*.shier`，那么`a.shier`、`b.shier`都能匹配

2. `#`：匹配0个或多个单词，比如`a.#`，那么`a.a`、`a.b`、`a.a.a`都匹配

   > `#.a.#` 可以匹配 `a.b`  、`a1.a`  等形式
   >
   > a的前面可以是0个或多个字符串，后面也是0个或者多个字符串

注意，这里的匹配和 MySQL 的 like 的 % 不一祥，只按照单词来匹配，每个`'.'`分隔单词，如果是`'#.'`，其实可以忽略，匹配0个词可以的

![image-20230620131840253](https://paperfly-blog.oss-cn-shenzhen.aliyuncs.com/blog/image-20230620131840253.png)

示例场景：

> 老板下发一个任务，让多个任务组都能接受到这个任务消息

![image-20230623145040023](https://paperfly-blog.oss-cn-shenzhen.aliyuncs.com/blog/image-20230623145040023.png)



生产者代码：

```java
/**
 * @author Shier
 * 主题消费者 - 生产者
 */
public class TopicProducer {

  private static final String TOPIC_EXCHANGE = "topic-exchange";

  public static void main(String[] argv) throws Exception {
    ConnectionFactory factory = new ConnectionFactory();
        // 设置 rabbitmq 对应的信息
        factory.setHost("xxxxx.xxxx.xxxx");
        factory.setUsername("xxxx.xxxx.xxx");
        factory.setPassword("xxx.xxx.xxx");

    try (Connection connection = factory.newConnection();
         Channel channel = connection.createChannel()) {

        channel.exchangeDeclare(TOPIC_EXCHANGE, "topic");

        Scanner scanner = new Scanner(System.in);
        while (scanner.hasNext()) {
            String userInput = scanner.nextLine();
            String[] splits = userInput.split(" ");
            if (splits.length < 1) {
                continue;
            }
            String message = splits[0];
            String routingKey = splits[1];

            channel.basicPublish(TOPIC_EXCHANGE, routingKey, null, message.getBytes("UTF-8"));
            System.out.println(" [x] Sent '" + message + " with routing " + routingKey + "'");
        }
    }
  }
}
```

消费者代码：

```java
/**
 * @author Shier
 * 主题交换机 - 消费者
 */
public class TopicConsumer {

    private static final String TOPIC_EXCHANGE = "topic-exchange";

    public static void main(String[] argv) throws Exception {
        ConnectionFactory factory = new ConnectionFactory();
        // 设置 rabbitmq 对应的信息
        factory.setHost("xxxxx.xxxx.xxxx");
        factory.setUsername("xxxx.xxxx.xxx");
        factory.setPassword("xxx.xxx.xxx");

        Connection connection = factory.newConnection();
        Channel channel = connection.createChannel();

        channel.exchangeDeclare(TOPIC_EXCHANGE, "topic");

        // 创建前端队列，分配一个队列名称：frontend
        String queueName = "frontend_queue";
        channel.queueDeclare(queueName, true, false, false, null);
        channel.queueBind(queueName, TOPIC_EXCHANGE, "#.前端.#");

        // 创建后端队列，分配一个队列名称：backend
        String queueName2 = "backend_queue";
        channel.queueDeclare(queueName2, true, false, false, null);
        channel.queueBind(queueName2, TOPIC_EXCHANGE, "#.后端.#");

        // 创建后端队列，分配一个队列名称：backend
        String queueName3 = "product_queue";
        channel.queueDeclare(queueName3, true, false, false, null);
        channel.queueBind(queueName3, TOPIC_EXCHANGE, "#.产品.#");

        System.out.println(" [*] Waiting for messages. To exit press CTRL+C");

        // A队列监听机制
        DeliverCallback xiaoADeliverCallback = (consumerTag, delivery) -> {
            String message = new String(delivery.getBody(), "UTF-8");
            System.out.println(" [xiaoA] Received '" +
                    delivery.getEnvelope().getRoutingKey() + "':'" + message + "'");
        };
        // B队列监听机制
        DeliverCallback xiaoBDeliverCallback = (consumerTag, delivery) -> {
            String message = new String(delivery.getBody(), "UTF-8");
            System.out.println(" [xiaoB] Received '" +
                    delivery.getEnvelope().getRoutingKey() + "':'" + message + "'");
        };

        // C队列监听机制
        DeliverCallback xiaoCDeliverCallback = (consumerTag, delivery) -> {
            String message = new String(delivery.getBody(), "UTF-8");
            System.out.println(" [xiaoC] Received '" +
                    delivery.getEnvelope().getRoutingKey() + "':'" + message + "'");
        };

        channel.basicConsume(queueName, true, xiaoADeliverCallback, consumerTag -> {
        });
        channel.basicConsume(queueName2, true, xiaoBDeliverCallback, consumerTag -> {
        });
        channel.basicConsume(queueName3, true, xiaoCDeliverCallback, consumerTag -> {
        });
    }
}
```



#### 3.4.5 Headers 交换机

类以主题和直接交换机，可以根据headers中的内容来指定发送到哪个队列。

由于性差、比较复杂，一般不推荐使用。

> rabbitmq中的 headers 交换机是什么？有什么用？什么场景下会用？有什么优缺点？
>
> AI答：
>
> 在RabbitMQ消息队列中，Headers交换机是一种特殊类型的交换机，它使用消息头（Headers）中的键值对来进行消息路由。与其他类型的交换机不同，Headers交换机无视消息的路由键，而是根据消息头中的键值对来决定消息的路由。
>
> Headers交换机的主要用途是基于消息的附加属性来进行路由，而不是通过固定的路由键进行匹配。它适用于一些特殊的场景，例如需要根据消息的特定属性来进行过滤、路由或匹配的情况。Headers交换机可以让我们在消息发布的时候根据消息头的键值对来指定消息的目标队列或者路由规则。
>
> 优点：
>
> - 灵活性高：Headers交换机可以根据消息的附加属性进行灵活的路由和匹配，可以满足一些特殊需求。
> - 可以进行复杂的匹配：Headers交换机支持复杂的匹配规则，可以根据多个属性进行匹配和路由。
>
> 缺点：
>
> - 性能相对较低：Headers交换机需要对所有消息的消息头进行匹配，因此在消息数量较大的情况下性能会有所下降。
> - 配置复杂：Headers交换机需要明确指定消息头的键值对作为匹配条件，因此在配置时相对较为复杂。
>
> 在实际应用中，Headers交换机常常用于一些特定的场景，例如：
>
> - 根据消息的自定义属性进行过滤和路由，以达到消息的分发和消费的目的。
> - 对消息进行定制化的路由，根据消息头中的自定义属性选择不同的消费者进行处理。
>
> 需要注意的是，Headers交换机并不是常见的交换机类型，一般情况下，使用Direct、Fanout和Topic交换机就可以满足大多数的需求。只有在需要根据消息的附加属性进行路由或过滤时才考虑使用Headers交换机。



#### 3.4.6 RPC：远程过程调用

> 支持用消息队列来模拟RPC的调用，但是一般没必要，直接用Dubbo、GRPC等RPC框架就好了。
>
> 
>
> 实现一个场景总有更合适的、更专注的技术。一般都是直接使用Dubbo、GRPC等RPC框架技术即可。

在RabbitMQ消息队列中，RPC（Remote Procedure Call，远程过程调用）是一种通信模式，用于实现分布式系统中的远程调用。RPC允许一个进程（称为客户端）调用另一个进程（称为服务端）上的某个函数并获取返回结果，就像在本地调用函数一样。

RPC的使用可以简化分布式系统间的通信，提供更方便的远程调用方式。客户端可以将请求参数封装成消息发送到消息队列的一个特定队列中，服务端监听该队列并接收消息，然后执行相应的处理逻辑，并将处理结果发送回客户端。

使用RPC的主要目的是实现分布式系统的协同工作，例如：

- 将计算任务分发到不同的节点上进行并行处理，提高系统的性能和响应速度。
- 实现微服务架构中的服务间的函数调用。

优点：

- 解耦性：RPC允许服务端和客户端通过异步消息传递进行通信，减少了服务之间的直接依赖。
- 可扩展性：RPC可以方便地加入新的服务或者移除不再需要的服务，通过消息队列可以实现动态的服务发现和注册。
- 并发处理：RPC可以实现并发处理多个请求，提高系统的吞吐量和并发能力。

缺点：

- 性能开销：RPC通过网络进行通信，相对于本地函数调用会有一定的性能开销。
- 配置复杂：RPC需要对客户端和服务端进行正确的配置和协调，包括队列、交换机、路由等参数的设置。

常见的使用场景包括：

- 分布式系统中的服务间调用
- 实现远程方法调用，比如客户端通过RPC调用服务器端的API

在选择使用RPC时，需要考虑系统的性能需求和规模，确保消息队列的性能能够满足RPC的通信需求。同时，还需要合理设置RPC的超时和重试机制，以应对网络故障或服务不可用的情况。



### 3.5 Rabbit MQ 核心特性

#### 3.5.1 消息过期机制

官网：https://www.rabbitmq.com/ttl.html

可以给每条消息指定一个有效期，一段时间内未被消费者处理，就过期了。

示例场景：

- 消费者（库存系统）挂了，一个订单15分钟还没被库存系统处理，这个订单其实已经失效了，哪怕库存系统再恢复，其实也不用扣减库存。

适用场景：清理过期数据、模拟延迟队列的实现（不开会员就慢速）、专门让某个程序处理过期请求

1. 订单超时取消：在电商平台中，可以设置订单的过期时间，如果用户在规定时间内未支付订单，则将订单标记为过期取消。
2. 预约失效处理：在医院或美容院等场所，用户预约服务后，可以设置预约消息的过期时间，如果用户在规定时间内未到达，可以取消预约或释放时间。
3. 缓存更新和失效：在网站或应用中，可以将数据加载到缓存中，设置过期时间以保持数据的新鲜度，并在数据过期后重新加载最新数据。
4. 日志记录和清理：在系统中，可以将日志记录为消息发送到队列中，并设置过期时间以限制日志数据的保留时间，以及自动清理过期的日志。
5. 定时任务调度：可以使用RabbitMQ结合过期时间来实现定时任务调度。通过设置消息的过期时间，将任务消息发送到队列中，并在过期后触发执行相应的任务操作。

> AI 说：
>
> RabbitMQ提供了消息过期机制（Message Expiration）作为其核心特性之一。该机制允许你为消息设置一个过期时间，在此时间后，如果消息仍然未被消费者消费，将会自动从队列中删除。
>
> 消息过期机制的主要用途是处理一些具有时间敏感性的消息，例如临时的状态更新、时效性较强的任务等。通过设置消息的过期时间，可以确保消息在一定时间内得到及时消费，避免因消息堆积而导致的资源浪费或业务延迟。
>
> 优点：
>
> - 灵活性高：可以根据消息的特定需求，灵活地设置消息的过期时间，以确保消息的时效性。
> - 资源利用率高：过期的消息会被自动从队列中删除，避免了无效消息的堆积和影响队列性能。
>
> 缺点：
>
> - 对延迟要求较高：在消息过期之前，需要确保消费者能够及时消费到消息，避免消息过期而导致业务延迟。
> - 不适用于持久化：如果消息是持久化的（persistent），则即使消息过期了，也不会自动从磁盘上删除。
>
> 使用消息过期机制的常见场景包括：
>
> - 定时任务：在消息中设置过期时间，确保任务在规定时间内被消费者消费。
> - 缓存管理：对于一些缓存数据，可以设置过期时间来自动清理过期的数据。
> - 临时状态更新：例如在某个状态更新后一段时间内没有被消费者消费，则可以设置消息过期，避免无效状态的堆积。
>
> 值得注意的是，消息过期机制适用于一些具有时效性需求的场景。如果不需要强制实现消息的时效性，可以不使用消息过期机制，而是使用其他更适合的机制来处理消息的过期和清理，比如定期清理无效消息等。

1）给队列中的所有消息指定过期时间

```java
// 创建队列
Map<String, Object> msg = new HashMap<String, Object>();
msg.put("x-message-ttl", 5000);
// 指定args参数
channel.queueDeclare(TTL_QUEUE, false, false, false, msg);
```

如果在过期时间内，还没有消费者取消息，消息才会过期。

注意，如果消息已经接收到，但是没确认，是不会过期的。

> 如果消息处于待消费状态并且过期时间到达后，消息将被标记为过期。但是，如果消息已经被消费者消费，并且正在处理过程中，即使过期时间到达，消息仍然会被正常处理。

生产者代码：

```java
/**
 * @author Shier
 */
public class TtlProducer {
    private final static String TTL_QUEUE = "ttl-queue";

    public static void main(String[] argv) throws Exception {
        // 创建链接工厂
        ConnectionFactory factory = new ConnectionFactory();
        // 设置 rabbitmq 对应的信息
        factory.setHost("xxxxx.xxxx.xxxx");
        factory.setUsername("xxxx.xxxx.xxx");
        factory.setPassword("xxx.xxx.xxx");
        try (Connection connection = factory.newConnection();
             // 建立链接，创建频道
             Channel channel = connection.createChannel()) {
            // 创建消息队列 要删除掉，因为已经在消费者中创建了队列，没有必要再重新创建一次这个队列，如果在此处还创建队列，里面的参数必须要和消费者的参数一致
            // channel.queueDeclare(SINGLE_QUEUE_NAME, false, false, false, null);
            String message = "Hello World!";
            channel.basicPublish("", TTL_QUEUE, null, message.getBytes(StandardCharsets.UTF_8));
            System.out.println(" [x] Sent '" + message + "'");
        }
    }
}
```

消费者代码：

```java
/**
 * @author Shier
 */
public class TtlConsumer {

    private final static String TTL_QUEUE = "ttl-queue";

    public static void main(String[] argv) throws Exception {
        // 建立链接，
        ConnectionFactory factory = new ConnectionFactory();
        // 设置 rabbitmq 对应的信息
        factory.setHost("xxxxx.xxxx.xxxx");
        factory.setUsername("xxxx.xxxx.xxx");
        factory.setPassword("xxx.xxx.xxx");

        Connection connection = factory.newConnection();
        Channel channel = connection.createChannel();

        // 创建队列
        Map<String, Object> msg = new HashMap<String, Object>();
        msg.put("x-message-ttl", 5000);
        // 指定args参数
        channel.queueDeclare(TTL_QUEUE, false, false, false, msg);

        System.out.println(" [*] Waiting for messages. To exit press CTRL+C");

        // 定义如何处理消息
        DeliverCallback deliverCallback = (consumerTag, delivery) -> {
            String message = new String(delivery.getBody(), StandardCharsets.UTF_8);
            System.out.println(" [x] Received '" + message + "'");
        };
        // 消费消息 autoAck设置为false 取消掉自动确认消息
        channel.basicConsume(TTL_QUEUE, false, deliverCallback, consumerTag -> { });
    }
}
```



2）给某条消息指定过期时间

> 在消息发送者设置，也就是生产者代码处

语法

```java
byte[] messageBodyBytes = "Hello, world!".getBytes();
AMQP.BasicProperties properties = new AMQP.BasicProperties.Builder()
                                   .expiration("60000")
                                   .build();
channel.basicPublish("my-exchange", "routing-key", properties, messageBodyBytes);
```

示例代码：

```java
/**
 * @author Shier
 */
public class TtlProducer {
    private final static String TTL_QUEUE = "ttl-queue";

    public static void main(String[] argv) throws Exception {
        // 创建链接工厂
        ConnectionFactory factory = new ConnectionFactory();
        // 设置 rabbitmq 对应的信息
        factory.setHost("xxxxx.xxxx.xxxx");
        factory.setUsername("xxxx.xxxx.xxx");
        factory.setPassword("xxx.xxx.xxx");
        try (Connection connection = factory.newConnection();
             // 建立链接，创建频道
            Channel channel = connection.createChannel()) {
            String message = "Hello World!";
            AMQP.BasicProperties properties = new AMQP.BasicProperties.Builder()
                    .expiration("60000")
                    .build();
            channel.basicPublish("my-exchange", "routing-key", properties, message.getBytes(StandardCharsets.UTF_8));
            System.out.println(" [x] Sent '" + message + "'");
        }
    }
}
```





#### 3.5.2 消息确认机制（面试爱问）

官网介绍：https://www.rabbitmq.com/confirms.html

```java
channel.basicConsume(TASK_QUEUE_NAME, false, deliverCallback, consumerTag -> {});
// basicConsume()方法源码：
String basicConsume(String queue, boolean autoAck, DeliverCallback deliverCallback, CancelCallback cancelCallback) throws IOException;
```

为了保证消息成功被消费（快递成功被取走），rabbitmg提供了消息确认机制，当消费者接收到消息后，比如要给一个反馈：

- ack：消费成功（成功取到快递，我已确认收货，我的消息不用在保存在消息队列）
- nack：消费失败（未能去到快递，可能出于快递消失了、被人拿走了、快递存在问题等等的原因，我需要退货，需要和商家协调，你不能就此将我的消息去除）
- reject：拒绝

> 如果告诉 Rabbit MQ 服务器消费成功，服务器才会放心地移除消息。
>
> 支持配置autoack，会自动执行 ack 命令，接收到消息立刻就成功了。
>
> 
>
> 一般建议 autoack 改为 false ，根据实际情况，去手动确认。
>
> 比如：false时，就要自己手动的签收快递，等我收到快递之后，查看没有问题了，再去签收快递，这样可以按照流程减少不必要的事情。
>
> 		   true时，就是快递员不通知你就帮你签收了快递。（万一快递出了问题就不好搞了，不过现实中帮你签收了的情况也不少，有问题也还是可以处理的）

**指定确认某条消息**：

```java
channel.basicAck(delivery.getEnvelope().getDeliveryTag(),false);
// 源码
void basicAck(long deliveryTag, boolean multiple) throws IOException;
```

> 参数解释：
>
> 1. `deliveryTag`：表示要确认的消息的标识符。每个消息都有一个唯一的`deliveryTag`，用于标识消息的顺序。
> 2. `multiple`：表示是否批量确认消息。如果设置为`true`，则表示确认所有在`deliveryTag`之前的未确认消息；如果设置为`false`，则只确认当前`deliveryTag`的消息。
>
> 第二参数 multiple 批量确认：是指是否要一次性确认所有的历史消息直到当前这条

**指定拒绝某条消息**：

```java
channel.basicNack(delivery.getEnvelope().getDeliveryTag(),false,false);
// 源码
void basicNack(long deliveryTag, boolean multiple, boolean requeue)
            throws IOException;
```

> 参数解释：
>
> 1. `deliveryTag`：表示要否定确认的消息的标识符。每个消息都有一个唯一的`deliveryTag`，用于标识消息的顺序。
> 2. `multiple`：表示是否批量否定确认消息。如果设置为`true`，则表示否定所有在`deliveryTag`之前的未确认消息；如果设置为`false`，则只否定当前`deliveryTag`的消息。
> 3. `requeue`：表示是否将消息重新放回队列。如果设置为`true`，则消息将被重新放回队列并可以被其他消费者重新消费；如果设置为`false`，则消息将会被丢弃。
>
> 第三个参数表示是否重新入队，可用于重试





#### 3.5.3 死信队列

官网介绍：https://www.rabbitmq.com/dlx.html

为了保证消息的可靠性，比如每条消息都成功消费，需要提供一个容措机制，即：失败的消息怎么处理？

死信：过期的消息、拒收的消息、消息队列满了、处理失败的消息的統称

死信队列：专门处理死信的队列（注意，它就是一个普通队列，只不过是专门用来处理死信的，你甚至可以理解这个队列的名称叫“死信队列”)

死信交换机：专门给死信队列转发消息的交换机（注意，它就是一个普通交换机，只不过是专门给死信队列发消息而已，理解为这个交换机的名称就叫“死信交换机”)。也存在路由绑定

死信可以通过死信交换机绑定到死信队列。

> AI说：
>
> RabbitMQ的死信队列（Dead-Letter Queue）是一种**用于处理无法被消费或者被拒绝的消息的特殊队列**。当消息满足一定条件时，例如消息被拒绝、消息过期或消息无法路由到目标队列时，可以将这些消息投递到死信队列中进行特殊处理。
>
> 死信队列的主要作用是处理无法正常消费的消息，可以实现以下功能：
>
> - 错误处理：将处理失败的消息转移到死信队列中，以便进行后续的异常处理、日志记录等。
> - 重新处理：将无法路由的消息转移到死信队列后，可以进行二次处理或重新投递。
>
> 优点：
>
> - 异常处理：可以对处理失败的消息进行特殊的异常处理，以便排查和修复错误。
> - 重试机制：死信队列可以用于重新处理失败的消息，可以实现消息的重试机制，提高消息的可靠性。
>
> 缺点：
>
> - 增加复杂性：使用死信队列增加了系统的复杂性，需要设计和实现针对死信队列的处理逻辑。
> - 需要额外的资源：死信队列需要额外的队列和处理逻辑，可能增加系统的资源消耗。
>
> 使用死信队列的常见场景包括：
>
> - 消息重试：当消息处理失败时，可以将消息投递到死信队列中进行重试。
> - 异常处理：对于无法正常处理的消息，可以将其转移到死信队列以进行特殊的异常处理。
> - 延迟队列：通过设置消息的过期时间以及将过期的消息投递到死信队列，可以模拟实现延迟队列的功能。
>
> 需要注意的是，死信队列的使用需要谨慎，不宜滥用。合理设置死信队列的条件和处理逻辑，以避免死信队列的堆积和对系统性能的影响。



示例场景：

![image-20230623211815329](https://paperfly-blog.oss-cn-shenzhen.aliyuncs.com/blog/image-20230623211815329.png)

实现过程：

1）创建死信交换机和死信队列，并且绑定关系

![image-20230623212927797](https://paperfly-blog.oss-cn-shenzhen.aliyuncs.com/blog/image-20230623212927797.png)

```java
// 创建老板的死信队列
String queueName = "laoban_queue";
channel.queueDeclare(queueName, true, false, false, null);
channel.queueBind(queueName, DLX_DIRECT_EXCHANGE, "laoban");

// 创建外包的死信队列
String queueName2 = "waibao_queue";
channel.queueDeclare(queueName2, true, false, false, null);
channel.queueBind(queueName2, DLX_DIRECT_EXCHANGE, "waibao");
```

2）给失败之后需要容错处理的的队列绑定死信交换机

示例代码：

```java
// 指定死信队列的参数
Map<String, Object> args = new HashMap<>();
// 指定死信队列绑定到哪个交换机 ，此处绑定的是 dlx-direct-exchange 交换机
args.put("x-dead-letter-exchange", DLX_DIRECT_EXCHANGE);
// 指定死信要转发到哪个死信队列，此处转发到 laoban 这个死信队列
args.put("x-dead-letter-routing-key", "laoban");

// 创建队列，分配一个队列名称：小红
String queueName = "xiaohong_queue";
channel.queueDeclare(queueName, true, false, false, args);
channel.queueBind(queueName, DIRECT_2_EXCHANGE, "xiaohong");

Map<String, Object> args1 = new HashMap<>();
args.put("x-dead-letter-exchange", DLX_DIRECT_EXCHANGE);
args.put("x-dead-letter-routing-key", "waibao");

// 创建队列，分配一个队列名称：小蓝
String queueName2 = "xiaolan_queue";
channel.queueDeclare(queueName2, true, false, false, args1);
channel.queueBind(queueName2, DIRECT_2_EXCHANGE, "xiaolan");
```

3）可以给要容错的队列指定死信之后的转发规则，死信应该再转发到哪个死信队列

```java
// 指定死信要转发到哪个死信队列，此处转发到 laoban 这个死信队列
args.put("x-dead-letter-routing-key", "laoban");
```

4）可以通过程序来读取死信队列中的消息，从而进行处理



死信队列生产者代码：

```java
/**
 * @author Shier
 */
public class DlxDirectProducer {

    private static final String DLX_DIRECT_EXCHANGE = "dlx-direct-exchange";
    private static final String WORK_EXCHANGE_NAME = "direct2-exchange";

    public static void main(String[] argv) throws Exception {
        ConnectionFactory factory = new ConnectionFactory();

        // 设置 rabbitmq 对应的信息
        factory.setHost("xxxxx.xxxx.xxxx");
        factory.setUsername("xxxx.xxxx.xxx");
        factory.setPassword("xxx.xxx.xxx");

        try (Connection connection = factory.newConnection();
             Channel channel = connection.createChannel()) {

            // 声明私信交换机
            channel.exchangeDeclare(DLX_DIRECT_EXCHANGE, "direct");

            // 创建老板的死信队列
            String queueName = "laoban_dlx_queue";
            channel.queueDeclare(queueName, true, false, false, null);
            channel.queueBind(queueName, DLX_DIRECT_EXCHANGE, "laoban");

            // 创建外包的死信队列
            String queueName2 = "waibao_dlx_queue";
            channel.queueDeclare(queueName2, true, false, false, null);
            channel.queueBind(queueName2, DLX_DIRECT_EXCHANGE, "waibao");


            // 老板队列监听机制
            DeliverCallback laobanDeliverCallback = (consumerTag, delivery) -> {
                String message = new String(delivery.getBody(), "UTF-8");
                // 拒绝消息
                channel.basicNack(delivery.getEnvelope().getDeliveryTag(),false,false);
                System.out.println(" [laoban] Received '" +
                        delivery.getEnvelope().getRoutingKey() + "':'" + message + "'");
            };
            // 外包队列监听机制
            DeliverCallback waibaoDeliverCallback = (consumerTag, delivery) -> {
                String message = new String(delivery.getBody(), "UTF-8");
                // 拒绝消息
                channel.basicNack(delivery.getEnvelope().getDeliveryTag(),false,false);
                System.out.println(" [waibao] Received '" +
                        delivery.getEnvelope().getRoutingKey() + "':'" + message + "'");
            };
            // 开启消费通道进行监听
            channel.basicConsume(queueName, false, laobanDeliverCallback, consumerTag -> {
            });
            channel.basicConsume(queueName2, false, waibaoDeliverCallback, consumerTag -> {
            });


            Scanner scanner = new Scanner(System.in);
            while (scanner.hasNext()) {
                String userInput = scanner.nextLine();
                String[] splits = userInput.split(" ");
                if (splits.length < 1) {
                    continue;
                }
                String message = splits[0];
                String routingKey = splits[1];

                channel.basicPublish(WORK_EXCHANGE_NAME, routingKey, null, message.getBytes("UTF-8"));
                System.out.println(" [x] Sent '" + message + " with routing " + routingKey + "'");
            }
        }
    }
}
```

死信队列消费者代码：

```java
/**
 * @author Shier
 */
public class DlxDirectConsumer {

    private static final String DLX_DIRECT_EXCHANGE = "dlx-direct-exchange";

    private static final String WORK_EXCHANGE_NAME = "direct2-exchange";

    public static void main(String[] argv) throws Exception {
        ConnectionFactory factory = new ConnectionFactory();
        // 设置 rabbitmq 对应的信息
        factory.setHost("xxxxx.xxxx.xxxx");
        factory.setUsername("xxxx.xxxx.xxx");
        factory.setPassword("xxx.xxx.xxx");

        Connection connection = factory.newConnection();
        Channel channel = connection.createChannel();

        channel.exchangeDeclare(WORK_EXCHANGE_NAME, "direct");

        // 指定死信队列的参数
        Map<String, Object> args = new HashMap<>();
        // 指定死信队列绑定到哪个交换机 ，此处绑定的是 dlx-direct-exchange 交换机
        args.put("x-dead-letter-exchange", DLX_DIRECT_EXCHANGE);
        // 指定死信要转发到哪个死信队列，此处转发到 laoban 这个死信队列
        args.put("x-dead-letter-routing-key", "laoban");

        // 创建队列，分配一个队列名称：小红
        String queueName = "xiaohong_queue";
        channel.queueDeclare(queueName, true, false, false, args);
        channel.queueBind(queueName, WORK_EXCHANGE_NAME, "xiaohong");

        Map<String, Object> args2 = new HashMap<>();
        args2.put("x-dead-letter-exchange", DLX_DIRECT_EXCHANGE);
        args2.put("x-dead-letter-routing-key", "waibao");

        // 创建队列，分配一个队列名称：小蓝
        String queueName2 = "xiaolan_queue";
        channel.queueDeclare(queueName2, true, false, false, args2);
        channel.queueBind(queueName2, WORK_EXCHANGE_NAME, "xiaolan");

        System.out.println(" [*] Waiting for messages. To exit press CTRL+C");

        // 小红队列监听机制
        DeliverCallback xiaohongDeliverCallback = (consumerTag, delivery) -> {
            String message = new String(delivery.getBody(), "UTF-8");
            // 拒绝消息
            channel.basicNack(delivery.getEnvelope().getDeliveryTag(),false,false);
            System.out.println(" [xiaohong] Received '" +
                    delivery.getEnvelope().getRoutingKey() + "':'" + message + "'");
        };
        // 小蓝队列监听机制
        DeliverCallback xiaolanDeliverCallback = (consumerTag, delivery) -> {
            String message = new String(delivery.getBody(), "UTF-8");
            // 拒绝消息
            channel.basicNack(delivery.getEnvelope().getDeliveryTag(),false,false);
            System.out.println(" [xiaolan] Received '" +
                    delivery.getEnvelope().getRoutingKey() + "':'" + message + "'");
        };
        // 开启消费通道进行监听
        channel.basicConsume(queueName, false, xiaohongDeliverCallback, consumerTag -> {
        });
        channel.basicConsume(queueName2, false, xiaolanDeliverCallback, consumerTag -> {
        });
    }
}
```



> 总得来说就是，老板下发任务给员工（消息进入员工队列），如果员工发现自己处理不了这个任务（员工将此消息交由死信交换机，然后叫老板帮忙解决一下），就会再去请求老板帮忙解决（最后将此消息放入到老板的死信队列中）
>
> 
>
> 以下是一个具体的例子来说明死信队列的使用和解决过程：
>
> 假设有一个订单支付系统，包含订单队列和支付结果队列。当用户下单后，订单信息会被发送到订单队列中进行处理，然后根据支付结果将订单信息发送到支付结果队列。但是有时订单队列中的消息处理失败或者超时，导致支付结果无法被发送到支付结果队列中。
>
> 在这种情况下，可以使用死信队列来解决问题。具体步骤如下：
>
> 1. 创建一个正常的订单队列和支付结果队列，并定义它们之间的交换机和绑定关系。
> 2. 创建一个死信交换机和死信队列，并定义它们之间的绑定关系。将订单队列的死信路由到死信交换机和死信队列中。
> 3. 在订单队列中设置消息的过期时间，当消息过期后，会成为死信消息，并被路由到死信交换机和死信队列中。
> 4. 在支付结果队列中消费消息，处理订单的支付结果。
> 5. 在死信队列中消费死信消息，并进行相应的处理。可以记录日志、发送通知等。
>
> 通过以上步骤，当订单队列中的消息不能按时处理时，消息会过期成为死信消息，并被路由到死信队列中，然后我们可以在死信队列中进行相应的处理。
>
> 例如，在支付结果队列中的消费者发现某个订单无法处理时，可以将该消息发送到死信交换机和死信队列中，然后我们可以在死信队列中写入日志或者发送邮件通知相关人员，以便进一步处理这些死信消息。
>
> 这样，通过使用死信队列，我们可以更好地处理无法被消费的消息，并进行相关的补救措施，从而提高系统的可靠性和容错性。

#### 3.5.4 延迟队列

官网地址：https://blog.rabbitmq.com/posts/2015/04/scheduling-messages-with-rabbitmq

延迟队列（Delayed Queue），它允许消息在一定的延迟时间后被消费。

作用：**在消息到达队列后，不立即将消息投递给消费者，而是在一定延迟时间后再进行投递。延迟队列通常用于需要延迟处理的业务场景。**  延迟队列主要用于处理需要在特定时间后执行的任务或延迟消息。它可以为消息设置一个延迟时间，在指定的延迟时间后，消息会被自动投递到指定的消费者。例如定时任务、消息重试、延迟通知等。



延迟队列适用于许多场景，包括：

1. 定时任务：可以使用延迟队列来实现任务的定时触发，例如定时发送邮件或推送通知。
2. 消息重试：当某个消息失败后，可以将其放入延迟队列，并设置延迟时间，以便稍后重新投递。
3. 延迟通知：例如在某个时间后发送提醒通知。

延迟队列的优点有：

1. 灵活性：可以根据实际需求，灵活地设置延迟时间，适应各种业务场景。
2. 解耦性：延迟队列可以将消息发送与消费解耦，提高系统的可扩展性和稳定性。
3. 可靠性：通过延迟队列，可以确保消息在一定时间后被投递，降低消息丢失的风险。

延迟队列的缺点有：

1. 系统复杂性：引入延迟队列会增加系统的复杂性和维护成本。
2. 延迟时间不准确：由于网络延迟、系统负载等原因，延迟时间可能会有一定的误差。



### 3.6 RabbitMQ重点知识

> 也是面试考点

1. 消息队列的概念、模型（上面的生产者、消费者图解）、应用场景
2. 交换机的类别、路由绑定的关系
3. 消息可靠性
   - 消息确认机制(ack、nack、reject)
   - 消息特久化(durable)
   - 消息过期机制
   - 死信队列
4. 延迟队列（类似死信队列）
5. 顺序消费、消费幂等性 
6. 可扩展性（仅作了解）
   - 集群
   - 故障的恢复机制
   - 镜像
7. 运维监控告警（仅作了解）

## 4、RabbitMQ BI 项目实战

### 4.1 选择客户端

> 怎么在项目中使用RabbitMQ?

1. 使用官方的客户端

优点：兼容性好，换语言成本低，比较灵活

缺点：太灵活，要自己去处理一些事情。比如要自己维护管理链接，很麻烦。

1. 使用封装好的客户端，比如Spring Boot RabbitMQ Starter

优点：简单易用，直接配置直接用，更方便地去管理连接

缺点：封装的太好了，你没学过的话反而不知道怎么用。不够灵活，被框架限制。

> 根据场景来选择，没有绝对的优劣，类似 jdbc 和 MyBatis
>
> 
>
> 优点：
>
> 1. 强大的功能支持：RabbitMQ Java客户端提供了丰富的功能和特性，例如消息确认、事务支持、消费者拉取、消息重试、可靠性消息传递等。
> 2. 可靠性和稳定性：通过使用可靠的消息确认机制和事务支持，Java客户端可以确保消息的可靠传递和处理。
> 3. 开发人员社区支持：RabbitMQ Java客户端拥有广泛的开发人员社区支持，并且有大量的文档和示例代码可供参考。
> 4. 良好的性能：Java客户端采用了高性能的网络通信协议和线程模型，可以在高并发的场景下表现出较好的性能。
>
> 缺点：
>
> 1. 复杂性：相对于其他语言的RabbitMQ客户端，Java客户端可能会更加复杂和臃肿，需要更多的代码和配置。
> 2. 内存占用：由于Java客户端的一些设计和依赖，可能会占用较多的内存资源，尤其在高并发的情况下。

本次使用 Spring Boot RabbitMQ Starter (因为我们是Spring Boot项目)

如果你有一定水平，有基础，英文好，建议看官方文档，不要看过期博客！

官网地址：https://spring.io/guides/gs/messaging-rabbitmq/



### 4.2 项目基础实战

#### 4.2.1 引入依赖

```
<!-- https://mvnrepository.com/artifact/org.springframework.boot/spring-boot-starter-amqp -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-amqp</artifactId>
    <version>2.7.2</version>
</dependency>
```

#### 4.2.2 在yml中进行配置

```yml
spring: 
    # rabbitmq 信息
    rabbitmq:
      host: xxxx
      password: kcsen123456
      username: shier
      port: 5672
```

#### 4.2.3 创建交换机和队列

```java
public class RabbitMqInitDemo {
    public static void main(String[] args) {
        try {
            ConnectionFactory factory = new ConnectionFactory();
            // 设置 rabbitmq 对应的信息
            factory.setHost("xxxxx.xxxx.xxxx");
            factory.setUsername("xxxx.xxxx.xxx");
            factory.setPassword("xxx.xxx.xxx");

            Connection connection = factory.newConnection();
            Channel channel = connection.createChannel();
            
            String demoExchange = "demo_exchange";
            
            channel.exchangeDeclare(demoExchange, "direct");

            // 创建队列，分配一个队列名称：小紫
            String queueName = "demo_queue";
            channel.queueDeclare(queueName, true, false, false, null);
            channel.queueBind(queueName, demoExchange, "demo_routingKey");
            
        }catch (Exception e){
            
        }
    }

}
```

#### 4.2.4 生产者代码

```java
@Component
public class RabbitMqMessageProducer {

    @Resource
    private RabbitTemplate rabbitTemplate;

    /**
     * 发送消息
     * @param exchange
     * @param routingKey
     * @param message
     */
    public void sendMessage(String exchange, String routingKey, String message) {
        rabbitTemplate.convertAndSend(exchange, routingKey, message);
    }
}
```

#### 4.2.5 消费者代码

```java
@Component
@Slf4j
public class RabbitMqMessageConsumer {

    /**
     * 指定程序监听的消息队列和确认机制
     * @param message
     * @param channel
     * @param deliveryTag
     */
    @RabbitListener(queues = {"demo_queue"}, ackMode = "MANUAL")
    private void receiveMessage(String message, Channel channel,@Header(AmqpHeaders.DELIVERY_TAG) long deliveryTag){
      log.info("receiveMessage = {}",message);
        try {
            // 手动确认消息
            channel.basicAck(deliveryTag,false);
        } catch (IOException e) {
            throw new RuntimeException(e);
        }
    }
}

```

#### 4.2.6 单元测试执行

```java
@SpringBootTest
class RabbitMqMessageProducerTest {


    @Resource
    private RabbitMqMessageProducer messageProducer;


    @Test
    void sendMessage() {
        messageProducer.sendMessage("demo_exchange","demo_routingKey","欢迎来到十二智能BI系统");
    }
}
```

结果如下：

![image-20230721204105992](https://paperfly-blog.oss-cn-shenzhen.aliyuncs.com/blog/image-20230721204105992.png)

### 4.3 BI项目改造

#### 4.3.1 改造前BI

以前是把任务提交到线程池，然后在线程池提交中编写处理程序的代码，线程池内排队。

如果程序中断了，任务就没了，就丢了。

####  4.3.2 改造后的BI

改造后的流程：

1. 把任务提交改为向队列发送消息

2. 写一个专引门的接受消息的程序，处理任务

3. 如果程序中断了，消息未被确认，还会重发

4. 现在，消息全部集中发到消息队列，可以部署多个后端，都从同一个地方取任务，从而实现了分布式负载均衡

实现步骤：

1. 创建交换机和队列
2. 将线程池中的执行代码移到消费者类中
3. 根据消费者的需求来确认消息的格式(chartld)
4. 将提交线程池改造为发送消息到队列

验证

验证发现，如果程序中断了，没有 ack、也没有 nack (服务中断，没有任何响应)，那么这条消息会被重新放到消息队列中，从而实现了每个任务都会执行。

#### 4.3.3 实现代码



1）BI 创建队列和交换机

```java
/**
 * 用于创建测试程序用到的交换机和队列（只用在程序启动前执行一次）
 */
public class BiInitMain {

    public static void main(String[] args) {
        try {
            ConnectionFactory factory = new ConnectionFactory();
            factory.setHost("localhost");
            Connection connection = factory.newConnection();
            Channel channel = connection.createChannel();
            String EXCHANGE_NAME =  BiMqConstant.BI_EXCHANGE_NAME;
            channel.exchangeDeclare(EXCHANGE_NAME, "direct");

            // 创建队列，随机分配一个队列名称
            String queueName = BiMqConstant.BI_QUEUE_NAME;
            channel.queueDeclare(queueName, true, false, false, null);
            channel.queueBind(queueName, EXCHANGE_NAME,  BiMqConstant.BI_ROUTING_KEY);
        } catch (Exception e) {

        }

    }
}

```

2）BI 消费者程序

第一步：创建消费者要执行的方法

```java
@SneakyThrows
@RabbitListener(queues = {BiMqConstant.BI_QUEUE_NAME}, ackMode = "MANUAL")
public void receiveMessage(String message, Channel channel, @Header(AmqpHeaders.DELIVERY_TAG) long deliveryTag)
```

有两个注解：

1. @SneakyThrows：Lombok 提供的注解，用于在方法中自动添加异常抛出代码。它可以帮助简化在方法中处理受检查异常的代码，不需要显式地编写 try-catch 语句。
2. @RabbitListener(queues = {BiMqConstant.BI_QUEUE_NAME}, ackMode = "MANUAL")

`@RabbitListener` 是 Spring AMQP 提供的注解，用于标记一个方法为 RabbitMQ 消息监听器。其中，`queues` 参数指定了要监听的队列名称，可以传入一个字符串数组表示监听多个队列。

> queues = {BiMqConstant.BI_QUEUE_NAME}` 表示该消息监听器将监听名为 `BiMqConstant.BI_QUEUE_NAME` 的队列。
>
> 
>
> ackMode` 参数用来指定消息确认模式。` 
>
> ackMode = "MANUAL"` 表示手动确认模式。这意味着当消费者成功处理一条消息后，需要显式调用 `channel.basicAck()` 方法进行消息确认；如果出现异常或处理失败，也需要调用 `channel.basicNack()` 方法进行消息拒绝。` 
>
> ackMode = "AUTO时"，`表示使用自动确认模式。在这种模式下，当消费者成功接收并处理一条消息后，RabbitMQ 会自动将该消息标记为已确认，不需要消费者显式地进行确认操作`。
>
> 手动确认模式相较于自动确认模式（`ackMode = "AUTO"`），能够更加精确地控制消息的确认或拒绝，从而实现更可靠的消息处理。



在RabbitMQ消息消费者方法的参数中，有三个参数：

1. `String message`：表示接收到的消息内容。这是你从消息队列中获取到的原始消息。
2. `Channel channel`：表示与RabbitMQ服务器之间的通信通道。通过该通道，你可以与RabbitMQ进行交互，如确认收到消息、拒绝消息等。
3. `@Header(AmqpHeaders.DELIVERY_TAG) long deliveryTag`：表示消息的传递标签。每个传送的消息都会被分配一个唯一的delivery tag，用于标识该消息。通过delivery tag，你可以在需要时对消息进行确认、拒绝、重新投递等操作。

在消费者方法中，通过使用这些参数，你可以根据实际业务需求来处理消息。例如，根据消息内容进行业务处理，然后使用channel进行消息确认或拒绝，并使用delivery tag标识要操作的具体消息。

第二步：处理接受到的消息内容

```java
if (StringUtils.isBlank(message)) {
    // 消息为空，则拒绝掉消息
    channel.basicNack(deliveryTag, false, false);
    throw new BusinessException(ErrorCode.PARAMS_ERROR, "接受到的消息为空");
}
```

`channel.basicNack(deliveryTag, false, false)` 是一个用于拒绝消息并通知 RabbitMQ 的方法。它的参数解释如下：

1. `deliveryTag`：表示要拒绝的消息的 delivery tag，即消息的唯一标识符。
2. `multiple`：指示是否拒绝多条消息。如果设置为 `true`，则会拒绝从当前消息起的所有未确认的消息；如果设置为 `false`，则仅拒绝当前消息。
3. `requeue`：指示是否将被拒绝的消息重新放回队列中。如果设置为 `true`，则消息会被重新放回队列，可以再次被消费；如果设置为 `false`，则消息会被直接丢弃。

通过调用 `channel.basicNack()` 方法，你可以拒绝某个特定的消息，并根据实际需求决定是否需要将消息重新放回队列。

需要注意的是，在使用 `basicNack()` 方法之前，你需要先确保已经设置了正确的 `ackMode` 为 `MANUAL`，否则该方法可能会抛出异常或产生不可预料的结果。

```java
@Component
@Slf4j
public class BiMessageConsumer {

    @Resource
    private BiChartService chartService;

    @Resource
    private AiManager aiManager;

    // 指定程序监听的消息队列和确认机制
    @SneakyThrows
    @RabbitListener(queues = {BiMqConstant.BI_QUEUE_NAME}, ackMode = "MANUAL")
    public void receiveMessage(String message, Channel channel, @Header(AmqpHeaders.DELIVERY_TAG) long deliveryTag) {
        log.info("receiveMessage message = {}", message);

        if (StringUtils.isBlank(message)) {
            // 如果失败，消息拒绝
            channel.basicNack(deliveryTag, false, false);
            throw new BusinessException(ErrorCode.SYSTEM_ERROR, "消息为空");
        }

        long chartId = Long.parseLong(message);
        BiChart chart = chartService.getById(chartId);
        if (chart == null) {
            channel.basicNack(deliveryTag, false, false);
            throw new BusinessException(ErrorCode.NOT_FOUND_ERROR, "图表为空");
        }

        // 先修改图表任务状态为 “执行中”。等执行成功后，修改为 “已完成”、保存执行结果；执行失败后，状态修改为 “失败”，记录任务失败信息。
        BiChart updateChart = new BiChart();
        updateChart.setId(chart.getId());
        updateChart.setChartStatus(ChartStatus.RUNNING.getValue());
        boolean b = chartService.updateById(updateChart);
        if (!b) {
            channel.basicNack(deliveryTag, false, false);
            chartService.handleChartUpdateError(chart.getId(), "更新图表执行中状态失败");
            return;
        }
        // 调用 AI
        String chartResult = aiManager.doChat(CommonConstant.BI_MODEL_ID, chartService.buildUserInput(chart));
        // 处理ai生成的数据
        OrganizeAiDataVo organizeAiData = chartService.getOrganizeAiData(chartResult);
        String genChart = organizeAiData.getGenChart();
        String genResult = organizeAiData.getGenResult();

        BiChart updateChartResult = new BiChart();
        updateChartResult.setId(chart.getId());
        updateChartResult.setGenChart(genChart);
        updateChartResult.setGenResult(genResult);
        updateChartResult.setChartStatus(ChartStatus.SUCCEED.getValue());
        boolean updateResult = chartService.updateById(updateChartResult);
        if (!updateResult) {
            channel.basicNack(deliveryTag, false, false);
            chartService.handleChartUpdateError(chart.getId(), "更新图表成功状态失败");
        }

        // 消息确认
        channel.basicAck(deliveryTag, false);
    }
}
```

3）生产者代码

```java
@Component
public class BiMessageProducer {

    @Resource
    private RabbitTemplate rabbitTemplate;

    /**
     * 发送消息
     * @param message
     */
    public void sendMessage(String message) {
        rabbitTemplate.convertAndSend(BiMqConstant.BI_EXCHANGE_NAME, BiMqConstant.BI_ROUTING_KEY, message);
    }

}

```

4）BiMq公共类

```java
public interface BiMqConstant {

    String BI_EXCHANGE_NAME = "bi_exchange";

    String BI_QUEUE_NAME = "bi_queue";

    String BI_ROUTING_KEY = "bi_routingKey";
}
```

实现 mq 的接口

```java
@Override
    public BiResponseVo getBiResponseAsyncMq(GenChartByAiRequest genChartByAiRequest, MultipartFile multipartFile) {
        String chartName = genChartByAiRequest.getChartName();
        String goal = genChartByAiRequest.getGoal();
        String chartType = genChartByAiRequest.getChartType();

        // 拿到当前登录用户信息
        UserInfoVO loginUser = userService.getUserLoginInfo();
        // 限流判断，每个用户一个限流器
        redisLimiterManager.doRateLimit("genChartByAi_" + loginUser.getUserId());

        String csvData = ExcelUtils.excelToCsv(multipartFile);
        String chartStatus = ChartStatus.WAIT.getValue();
        // 插入到数据库
        BiChart chart = BiChartUtils.getBiChartAsync(chartName, goal, chartType, csvData, chartStatus, loginUser);
        boolean saveResult = chartService.save(chart);
        ThrowUtils.throwIf(!saveResult, ErrorCode.SYSTEM_ERROR, "图表保存失败");

        long newChartId = chart.getId();
        biMessageProducer.sendMessage(String.valueOf(newChartId));
        BiResponseVo biResponse = new BiResponseVo();
        biResponse.setChartId(newChartId);
        return biResponse;
    }
```

### 4.4 mq添加死信队列和延迟时间

消息在某些情况下会变成死信（Dead Letter），比如：

1. 消息被消费者拒绝并且未设置重回队列：(NACK|| Reject)&&requeue== false
2. 消息过期
3. 队列达到最大长度，超过了Maxlength（消息数）或者 Maxlengthbytes（字节数），最先入队的消息会被发送到DLX。

照理说死信应该是被抛弃的，但是如果定义了死信交换机，当消息成为死信就会进入死信队列

- DLX：死信交换机（Dead Letter Exchange），队列在创建的时候可以指定一个，实际上也是普通的交换机
- DLQ：死信队列（DeadLetterQueue），被死信交换机绑定的队列，实际也是普通队列（例如替补球员也是普通球员）

#### 4.4.1 死信队列的使用

修改BiInitMain里面的内容：

1. 声明交换机的时候，搞个死信交换机

```java
// 创建一个通道
Channel channel = connection.createChannel();

String BI_EXCHANGE_NAME = BiMqConstant.BI_EXCHANGE_NAME;
String BI_EXCHANGE_DEAD = BiMqConstant.BI_EXCHANGE_DEAD;

// 声明一个直连交换机，一个死信交换机（其实就是普通交换机）
channel.exchangeDeclare(BI_EXCHANGE_NAME, "direct");
channel.exchangeDeclare(BI_EXCHANGE_DEAD, "direct");
```

2. 在声明正常队列时，通过设置 `x-message-ttl` 参数来指定消息的过期时间。使用 `Map<String, Object>` 对象来存储队列的属性，并将其作为参数传递给 `queueDeclare()` 方法。

```java
// 创建一个队列，随机分配一个队列名称
String queueName = BiMqConstant.BI_QUEUE_NAME;
// 通过设置 x-message-ttl 参数来指定消息的过期时间
Map<String, Object> queueArgs = new HashMap<>();
queueArgs.put("x-message-ttl", 60000); // 过期时间为 60 秒
// 参数解释：queueDeclare(String queue, boolean durable, boolean exclusive, boolean autoDelete, Map<String, Object> arguments)
// durable: 持久化队列（重启后依然存在）
// exclusive: 排他性队列（仅限此连接可见，连接关闭后队列删除）
// autoDelete: 自动删除队列（无消费者时自动删除）
channel.queueDeclare(queueName, true, false, false, queueArgs);
```

3. 为正常队列设置死信交换机，使过期消息变成死信后自动路由到私信交换机。

```java
String deadLetterRoutingKey = ""; // 空字符串，表示所有过期消息都会路由到死信交换机
Map<String, Object> deadArgs = new HashMap<>();
deadArgs.put("x-dead-letter-exchange", BI_EXCHANGE_DEAD);
deadArgs.put("x-dead-letter-routing-key", deadLetterRoutingKey);
// 将队列与交换机进行绑定
channel.queueBind(queueName, BI_EXCHANGE_NAME, BiMqConstant.BI_ROUTING_KEY,deadArgs);
```

4. 声明私信队列，并将其绑定到私信交换机。

```java
// 创建一个死信队列
String queueDeadName = BiMqConstant.BI_QUEUE_DEAD;
// 声明私信队列，并将其绑定到私信交换机。
channel.queueDeclare(queueDeadName,true,false,false,null);
channel.queueBind(queueDeadName,BI_EXCHANGE_DEAD,"");
```

完整代码：

执行前，先去管理界面把之前建好的交换机和队列删了

```java
/**
 * 用于创建测试程序用到的交换机和队列（只用在程序启动前执行一次）
 */
public class BiInitMain {

    public static void main(String[] args) {
        try {
            ConnectionFactory factory = new ConnectionFactory();
            factory.setHost("localhost");

            // 创建与 RabbitMQ 服务器的连接
            Connection connection = factory.newConnection();

            // 创建一个通道
            Channel channel = connection.createChannel();

            String BI_EXCHANGE_NAME = BiMqConstant.BI_EXCHANGE_NAME;
            String BI_EXCHANGE_DEAD = BiMqConstant.BI_EXCHANGE_DEAD;

            // 声明一个直连交换机，一个死信交换机（其实就是普通交换机）
            channel.exchangeDeclare(BI_EXCHANGE_NAME, "direct");
            channel.exchangeDeclare(BI_EXCHANGE_DEAD, "direct");

            // 创建一个队列，随机分配一个队列名称
            String queueName = BiMqConstant.BI_QUEUE_NAME;
            // 通过设置 x-message-ttl 参数来指定消息的过期时间
            Map<String, Object> queueArgs = new HashMap<>();
            queueArgs.put("x-message-ttl", 60000); // 过期时间为 60 秒
            // 参数解释：queueDeclare(String queue, boolean durable, boolean exclusive, boolean autoDelete, Map<String, Object> arguments)
            // durable: 持久化队列（重启后依然存在）
            // exclusive: 排他性队列（仅限此连接可见，连接关闭后队列删除）
            // autoDelete: 自动删除队列（无消费者时自动删除）
            channel.queueDeclare(queueName, true, false, false, queueArgs);

            String deadLetterRoutingKey = ""; // 空字符串，表示所有过期消息都会路由到死信交换机
            Map<String, Object> deadArgs = new HashMap<>();
            deadArgs.put("x-dead-letter-exchange", BI_EXCHANGE_DEAD);
            deadArgs.put("x-dead-letter-routing-key", deadLetterRoutingKey);
            // 将队列与交换机进行绑定
            channel.queueBind(queueName, BI_EXCHANGE_NAME, BiMqConstant.BI_ROUTING_KEY,deadArgs);

            // 创建一个死信队列
            String queueDeadName = BiMqConstant.BI_QUEUE_DEAD;
            // 声明私信队列，并将其绑定到私信交换机。
            channel.queueDeclare(queueDeadName,true,false,false,null);
            channel.queueBind(queueDeadName,BI_EXCHANGE_DEAD,"");

        } catch (Exception e) {
            // 处理异常情况
        }
    }
}
```

## 5. 添加redis缓存分页信息

### 5.1 缓存数据的设置

我们可以在客户端与数据库之间加上一个Redis缓存，先从Redis中查询，如果没有查到，再去MySQL中查询，同时查询完毕之后，将查询到的数据也存入Redis，这样当下一个用户来进行查询的时候，就可以直接从Redis中获取到数据

![img](https://paperfly-blog.oss-cn-shenzhen.aliyuncs.com/blog/6354a19216f2c2beb1b095dd.jpg)

#### 5.1.1 redis哪些数据结构可以缓存分页信息

在Redis中存储分页信息，可以使用以下几种数据结构：

1. Hash（哈希）：将每个用户的分页信息存储为一个Hash对象，其中键表示用户ID，字段表示页码，值表示对应页码的结果集。

   优点：可以方便地按用户和页码进行查找和更新，

   缺点：无法按照页码大小或排序进行查询。

   > redisTemplate.opsForHash().put()方法的参数如下所示：
   >
   > ```java
   > public void put(H key, HK hashKey, HV value);
   > ```
   >
   > - H：表示哈希对象（Hash），通常用于指定存储数据的键。
   > - HK：表示哈希字段（Hash Field），即在哈希对象中的字段，通常用于指定存储数据的子键。
   > - HV：表示哈希值（Hash Value），即要存储的数据。
   >
   > 具体含义如下：
   >
   > - key：表示哈希对象（Hash）的键。可以是任何类型的对象，但通常是一个字符串或其他可序列化的类型。
   > - hashKey：表示哈希字段（Hash Field）。可以是任何类型的对象，但通常是一个字符串或其他可序列化的类型。它用于唯一标识哈希对象中的每个字段。
   > - value：表示哈希值（Hash Value），即要存储的数据。可以是任何类型的对象。
   >
   > 例如，如果我们有一个哈希对象名为"user-page-info"，要将用户ID为"user1"的分页信息存储到该哈希对象中，页码为"1,2,3,4,5"，可以使用以下代码：
   >
   > ```java
   > redisTemplate.opsForHash().put("user-page-info", "user1", "1,2,3,4,5");
   > ```
   >
   > 这样就把"user1"作为哈希对象的字段，"1,2,3,4,5"作为对应字段的值存储在了"user-page-info"哈希对象中。

2. Sorted Set（有序集合）：将每个用户的分页信息存储为一个Sorted Set对象，其中键表示用户ID，成员表示页码，分值表示排序条件（如时间戳）。

   优点：可以根据分值进行排序和范围查询，适用于需要按照排序条件获取分页结果的场景，

   缺点：无法直接按用户进行查询。

   > redisTemplate.opsForZSet().add()方法的参数如下所示：
   >
   > ```java
   > public Boolean add(K key, V value, double score);
   > ```
   >
   > - K：表示有序集合（Sorted Set）的键。通常用于指定存储数据的键。
   > - V：表示有序集合的成员（Member），即要存储的数据。
   > - score：表示有序集合成员的分值（Score），用于确定成员在有序集合中的顺序。
   >
   > 具体含义如下：
   >
   > - key：表示有序集合的键。可以是任何类型的对象，但通常是一个字符串或其他可序列化的类型。
   > - value：表示有序集合的成员。可以是任何类型的对象。
   > - score：表示有序集合成员的分值。分值用于根据排序条件来确定成员在有序集合中的位置。分值必须为double类型。
   >
   > 例如，如果我们有一个有序集合名为"user-page-info"，要将用户ID为"user1"的分页信息存储到该有序集合中，同时指定排序分值为1，可以使用以下代码：
   >
   > ```java
   > redisTemplate.opsForZSet().add("user-page-info", "user1", 1);
   > ```
   >
   > 这样就将"user1"作为有序集合的一个成员，分值为1，存储在了"user-page-info"有序集合中。

3. List（列表）：将所有用户的分页信息存储为一个List对象，每个元素表示一页的结果集。

   优点：可以根据索引进行随机访问和分页查询，适用于需要根据页码进行快速定位的场景，

   缺点：无法按用户进行查询。

4. String（字符串）：将每个用户的分页信息以字符串形式存储在一个String对象中，可以将分页信息序列化为JSON字符串等格式。

   优点：简单易用，适用于较小的数据量和简单的查询需求，

   缺点：无法直接按用户或页码进行查询和更新。

根据redis的数据类型，我决定选择hash的方式，将key作为查询的唯一id，子健判断是否是需要的页面信息，value作为返回的对象

#### 5.1.2 缓存数据

引入redisTemplate进行使用

```java
@Resource
private RedisTemplate redisTemplate;

public PageResult<BiChart> getChartPage( @ParameterObject ChartQueryRequest chartQueryRequest) {
    int pageNum = chartQueryRequest.getPageNum();
    int pageSize = chartQueryRequest.getPageSize();
    
    Page<BiChart> result = chartService.page(new Page<>(pageNum, pageSize), getQueryWrapper(chartQueryRequest));

    Long userId = SecurityUtils.getUserId();
    // 每个用户的图表都是不一样的，所以拼接userId，就是唯一的
    String pageUser = CACHE_USER_PAGE+userId;
    // 根据需要查询的当前页码和每页大小拼接
    String userPageArg = CACHE_PAGE_ARG+pageNum+":"+pageSize;
    redisTemplate.opsForHash().put(pageUser, userPageArg, result);

    return PageResult.success(result);
}
```

这时候运行我们会报错，或者会出现redis里的数据是乱码的情况，这是因为redis存储数据的序列化问题

我们进行redisConfig的配置：

```java
@Configuration
public class RedisConfig {

    /**
     * 自定义 RedisTemplate
     * <p>
     * 修改 Redis 序列化方式，默认 JdkSerializationRedisSerializer
     *
     * @param redisConnectionFactory {@link RedisConnectionFactory}
     * @return {@link RedisTemplate}
     */
    @Bean
    public RedisTemplate<String, Object> redisTemplate(RedisConnectionFactory redisConnectionFactory) {

        RedisTemplate<String, Object> redisTemplate = new RedisTemplate<>();
        redisTemplate.setConnectionFactory(redisConnectionFactory);

        // 使用Jackson2JsonRedisSerializer作为序列化器
        Jackson2JsonRedisSerializer<Object> serializer = new Jackson2JsonRedisSerializer<>(Object.class);
        ObjectMapper objectMapper = new ObjectMapper();
        objectMapper.registerModule(new JavaTimeModule()); // 注册Java 8日期/时间模块
        serializer.setObjectMapper(objectMapper);

        redisTemplate.setDefaultSerializer(serializer);
        redisTemplate.setValueSerializer(serializer);
        redisTemplate.setHashValueSerializer(serializer);

        return redisTemplate;
    }
}
```

这样就解决啦，这时候我们去redis数据库看看信息

![image-20230722170357406](https://paperfly-blog.oss-cn-shenzhen.aliyuncs.com/blog/image-20230722170357406.png)

#### 5.1.3 读取缓存数据

这里会出现一个问题，就是读取数据的时候，返回的是一个Object，所以我们需要将Object转换为Page对象

我创建了一个redis工具类，这样可以多次复用

```java
public class RedisUtils {

    /**
     * 将对象转换成Page对象
     * @param obj
     * @param clazz
     * @return
     * @param <T>
     */
    @SuppressWarnings("unchecked")
    public static  <T> Page<T> convertToPage(Object obj, Class<T> clazz) {
        if (obj != null && obj instanceof Map) {
            Map<String, Object> resultMap = (Map<String, Object>) obj;
            if (resultMap.containsKey("records")) {
                Page<T> page = new Page<>();
                page.setRecords((List<T>) resultMap.get("records"));
                page.setCurrent(((Number) resultMap.getOrDefault("current", 0)).longValue());
                page.setSize(((Number) resultMap.getOrDefault("size", 0)).longValue());
                page.setTotal(((Number) resultMap.getOrDefault("total", 0)).longValue());

                // 可选：设置其他分页属性
                // page.setXXX(...)

                return page;
            }
        }
        return null;
    }
}
```

然后修改分页查询的方法：

```java
/**
 * 根据用户查询图表记录分页列表
 * @param chartQueryRequest
 * @return
 */
@Operation(summary = "用户查询记录分页列表", security = {@SecurityRequirement(name = "Authorization")})
@GetMapping("/page")
public PageResult<BiChart> getChartPage( @ParameterObject ChartQueryRequest chartQueryRequest) {
    int pageNum = chartQueryRequest.getPageNum();
    int pageSize = chartQueryRequest.getPageSize();

    Page<BiChart> result = chartService.page(new Page<>(pageNum, pageSize), getQueryWrapper(chartQueryRequest));

    Long userId = SecurityUtils.getUserId();
    // 每个用户的图表都是不一样的，所以拼接userId，就是唯一的
    String pageUser = CACHE_USER_PAGE+userId;
    // 根据需要查询的当前页码和每页大小拼接
    String userPageArg = CACHE_PAGE_ARG+pageNum+":"+pageSize;
    redisTemplate.opsForHash().put(pageUser, userPageArg, result);

    Object userPageInfoObj = redisTemplate.opsForHash().get(pageUser, userPageArg);
    // 将对象转换成Page对象
    Page<BiChart> cacheResult = RedisUtils.convertToPage(userPageInfoObj, BiChart.class);

    return PageResult.success(cacheResult);
}
```

如果没有报错就说明数据转换成功啦！

### 5.2 使用redis读取和缓存

#### 5.2.1 规范处理方法的位置

在controller层中将查询方法进行修改

```java
/**
 * 根据用户查询图表记录分页列表
 * @param chartQueryRequest
 * @return
 */
@Operation(summary = "用户查询记录分页列表", security = {@SecurityRequirement(name = "Authorization")})
@GetMapping("/page")
public PageResult<BiChart> getChartPage( @ParameterObject ChartQueryRequest chartQueryRequest) {
    if (chartQueryRequest == null){
        throw new BusinessException(ErrorCode.PARAMS_ERROR);
    }
    Page<BiChart> result = chartService.getChartPageByRedis(chartQueryRequest);

    return PageResult.success(result);
}
```

Service层添加了两个方法

```java
/**
 * 根据用户从redis缓存查询分页信息
 * @param chartQueryRequest
 * @return
 */
Page<BiChart> getChartPageByRedis(ChartQueryRequest chartQueryRequest);

/**
 * 获取查询包装类
 *
 * @param chartQueryRequest
 * @return
 */
QueryWrapper<BiChart> getQueryWrapper(ChartQueryRequest chartQueryRequest);
```

impl层实现这两个方法，并且实现缓存数据

```java
@Override
public Page<BiChart> getChartPageByRedis(ChartQueryRequest chartQueryRequest) {
    int pageNum = chartQueryRequest.getPageNum();
    int pageSize = chartQueryRequest.getPageSize();

    Page<BiChart> result = chartService.page(new Page<>(pageNum, pageSize), getQueryWrapper(chartQueryRequest));
    // 获取当前登录用户的Id
    Long userId = SecurityUtils.getUserId();

    // 每个用户的图表都是不一样的，所以拼接userId，就是唯一的
    String pageUser = CACHE_USER_PAGE+userId;
    // 根据需要查询的当前页码和每页大小拼接
    String userPageArg = CACHE_PAGE_ARG+pageNum+":"+pageSize;
    Object userPageInfoObj =  redisTemplate.opsForHash().get(pageUser, userPageArg);
	// 如果数据不存在，则先存进redis，再调用缓存
    if (userPageInfoObj == null){
        redisTemplate.opsForHash().put(pageUser, userPageArg, result);
        userPageInfoObj =  redisTemplate.opsForHash().get(pageUser, userPageArg);
    }
    // 将对象转换成Page对象
    Page<BiChart> cacheResult = RedisUtils.convertToPage(userPageInfoObj, BiChart.class);
    return cacheResult;
}

@Override
public QueryWrapper<BiChart> getQueryWrapper(ChartQueryRequest chartQueryRequest) {
    QueryWrapper<BiChart> queryWrapper = new QueryWrapper<>();
    if (chartQueryRequest == null) {
        return queryWrapper;
    }
    Long id = chartQueryRequest.getId();
    String chartName = chartQueryRequest.getChartName();
    String goal = chartQueryRequest.getGoal();
    String chartType = chartQueryRequest.getChartType();
    Long userId = chartQueryRequest.getUserId();

    // 获取当前用户id
    Long curUserId = SecurityUtils.getUser().getUserId();

    // 声明排序字段和排序方式
    String sortField = "create_time";
    String sortOrder = CommonConstant.SORT_ORDER_ASC;

    queryWrapper.eq(id != null && id > 0, "id", id);
    queryWrapper.like(StringUtils.isNotBlank(chartName), "chartName", chartName);
    queryWrapper.eq(StringUtils.isNotBlank(goal), "goal", goal);
    queryWrapper.eq(StringUtils.isNotBlank(chartType), "chartType", chartType);
    queryWrapper.eq(ObjectUtils.isNotEmpty(userId), "userId", userId);
    queryWrapper.eq(userId != null, "userId", userId);

    // 添加当前用户id的限制
    queryWrapper.eq(curUserId != null, "userId", curUserId);

    // 添加排序条件
    queryWrapper.orderBy(true, sortOrder == CommonConstant.SORT_ORDER_DESC, sortField);

    return queryWrapper;
}
```

### 5.3 解决缓存穿透问题

#### 5.3.1 缓存穿透问题的解决思路

- `缓存穿透`：缓存穿透是指客户端请求的数据在缓存中和数据库中都不存在，这样缓存永远都不会生效（只有数据库查到了，才会让redis缓存，但现在的问题是查不到），会频繁的去访问数据库。

<img src="https://paperfly-blog.oss-cn-shenzhen.aliyuncs.com/blog/image-20230722174639597.png" alt="image-20230722174639597" style="zoom:67%;" />

- 常见的结局方案有两种
  1. 缓存空对象
     - 优点：实现简单，维护方便
     - 缺点：额外的内存消耗，可能造成短期的不一致
  2. 布隆过滤
     - 优点：内存占用啥哦，没有多余的key
     - 缺点：实现复杂，可能存在误判
- `缓存空对象`思路分析：当我们客户端访问不存在的数据时，会先请求redis，但是此时redis中也没有数据，就会直接访问数据库，但是数据库里也没有数据，那么这个数据就穿透了缓存，直击数据库。但是数据库能承载的并发不如redis这么高，所以如果大量的请求同时都来访问这个不存在的数据，那么这些请求就会访问到数据库，简单的解决方案就是哪怕这个数据在数据库里不存在，我们也把这个这个数据存在redis中去（这就是为啥说会有`额外的内存消耗`），这样下次用户过来访问这个不存在的数据时，redis缓存中也能找到这个数据，不用去查数据库。可能造成的`短期不一致`是指在空对象的存活期间，我们更新了数据库，把这个空对象变成了正常的可以访问的数据，但由于空对象的TTL还没过，所以当用户来查询的时候，查询到的还是空对象，等TTL过了之后，才能访问到正确的数据，不过这种情况很少见罢了
- `布隆过滤`思路分析：布隆过滤器其实采用的是哈希思想来解决这个问题，通过一个庞大的二进制数组，根据哈希思想去判断当前这个要查询的数据是否存在，如果布隆过滤器判断存在，则放行，这个请求会去访问redis，哪怕此时redis中的数据过期了，但是数据库里一定会存在这个数据，从数据库中查询到数据之后，再将其放到redis中。如果布隆过滤器判断这个数据不存在，则直接返回。这种思想的优点在于节约内存空间，但存在误判，误判的原因在于：布隆过滤器使用的是哈希思想，只要是哈希思想，都可能存在哈希冲突

#### 5.3.2 使用缓存空对象解决问题

1. 如果获取到的缓存数据为空字符串，抛出自定义异常ErrorCode.NOT_FOUND_ERROR。

2. 如果获取的缓存数据为空，则执行数据库查询操作。

   1. 如果查询结果为空，设置缓存数据为空字符串，并设置过期时间为5分钟，并抛出自定义异常ErrorCode.NOT_FOUND_ERROR。

   2. 如果查询结构不为空，将查询结果存储到缓存中，并设置过期时间为5分钟。

3. 再次从Redis中获取用户缓存数据。

4. 将获取到的缓存数据转换成Page对象并返回。

```java
public Page<BiChart> getChartPageByRedis(ChartQueryRequest chartQueryRequest) {
    int pageNum = chartQueryRequest.getPageNum();
    int pageSize = chartQueryRequest.getPageSize();
    // 获取当前登录用户的Id
    Long userId = SecurityUtils.getUserId();

    // 每个用户的图表都是不一样的，所以拼接userId，就是唯一的
    String pageUser = CACHE_USER_PAGE+userId;
    // 根据需要查询的当前页码和每页大小拼接
    String userPageArg = CACHE_PAGE_ARG+pageNum+":"+pageSize;
    Object userPageInfoObj =  redisTemplate.opsForHash().get(pageUser, userPageArg);
    
    if ("".equals(userPageInfoObj)){
        throw new BusinessException(ErrorCode.NOT_FOUND_ERROR);
    }

    if (userPageInfoObj == null){
        // 执行查询数据库操作
        Page<BiChart> result = chartService.page(new Page<>(pageNum, pageSize), getQueryWrapper(chartQueryRequest));

        if (result.getRecords().isEmpty()) {
            // 如果查询结果为空，则设置一个空字符串，避免频繁查询数据库
            redisTemplate.opsForHash().put(pageUser, userPageArg, "");
            // 设置过期时间，防止缓存在长时间内一直为空集合而无法更新
            redisTemplate.expire(pageUser, CACHE_MINUTES_FIVE, TimeUnit.MINUTES);
            throw new BusinessException(ErrorCode.NOT_FOUND_ERROR);
        }
        // 存储结果到缓存中
        redisTemplate.opsForHash().put(pageUser, userPageArg, result);
        redisTemplate.expire(pageUser, CACHE_MINUTES_FIVE, TimeUnit.MINUTES);
        userPageInfoObj =  redisTemplate.opsForHash().get(pageUser, userPageArg);
    }

    // 将对象转换成Page对象
    Page<BiChart> cacheResult = RedisUtils.convertToPage(userPageInfoObj, BiChart.class);
    return cacheResult;
}
```

#### 5.3.3 使用布隆过滤器解决缓存穿透

第一步：数据预热

**出问题了！！！！！！**

对于分页缓存的预热，由于`pageNum`和`pageSize`是会变化的，所以无法提前将所有可能的查询结果都加载到缓存中。

所以使用布隆过滤器并不适用于这个场景，如果定死了页面查询的`pageNum`和`pageSize`那就还能够使用

下面就进行假设能进行消息预热的情况：

**假设能进行数据预热的情况**

首先，我这里在启动程序初始化redis的时候，就把需要的布隆过滤器名称创建好，所以在redisConfig里添加这个内容

```java
@Bean
public RBloomFilter<Object> myChartRBloom() {
    RedissonClient redissonClient = Redisson.create();
    RBloomFilter<Object> myChartRBloom = redissonClient.getBloomFilter("MyChart");
    if (!myChartRBloom.isExists()) {
        myChartRBloom.tryInit(10000, 0.01);
    }
    return myChartRBloom;
}
```

第二步：写好预热数据的方法

略。。。。。

第三步：实现功能

```java
/**
 * redis缓存分页查询，使用布隆过滤器解决缓存穿透问题
 * @param chartQueryRequest
 * @return
 */
@Override
public Page<BiChart> getChartPageByRedisBl(ChartQueryRequest chartQueryRequest) {
    int pageNum = chartQueryRequest.getPageNum();
    int pageSize = chartQueryRequest.getPageSize();
    // 获取当前登录用户的Id
    Long userId = SecurityUtils.getUserId();

    // 每个用户的图表都是不一样的，所以拼接userId，就是唯一的
    String pageUser = CACHE_USER_PAGE+userId;
    // 根据需要查询用户加上当前页码和每页大小拼接
    String userPageArg = CACHE_PAGE_ARG+userId+":"+pageNum+":"+pageSize;

    // 构造（获取）布隆过滤器对象
    RBloomFilter<Object> myChartRBloom = Redisson.create().getBloomFilter("MyChart");

    // 先通过布隆过滤器判断该查询是否存在于缓存中
    boolean existInCache = myChartRBloom.contains(userPageArg);
    if (!existInCache) {
        throw new BusinessException(ErrorCode.NOT_FOUND_ERROR);
    }

    // 先从缓存中获取数据
    Object userPageInfoObj = redisTemplate.opsForHash().get(pageUser, userPageArg);

    // 将对象转换成Page对象
    Page<BiChart> cacheResult = RedisUtils.convertToPage(userPageInfoObj, BiChart.class);
    return cacheResult;
}
```

如果不存在就直接报错，存在就去缓存拿数据

### 5.4 解决redis和mysql双写一致问题

#### 5.4.1 解决方案

在使用Redis和MySQL进行双写时，可能会面临数据一致性的问题。因为两个数据库是独立的，无法保证数据的同步更新。

以下是一些解决Redis和MySQL双写一致性问题的常见方法：

1. 异步双写：在业务流程中，先将数据写入MySQL数据库，然后再异步将数据写入Redis缓存。这种方式可以减小对数据库写操作的延迟，并提高响应速度。但是在读取数据时，可能会出现缓存与数据库之间的不一致。
2. 同步双写：在业务流程中，在数据写入MySQL数据库后，立即将数据同步写入Redis缓存。这样可以保证缓存和数据库之间的数据一致性。但是由于需要等待Redis写操作完成，可能会增加请求的响应时间。
3. 数据失效策略：在双写场景中，可以根据实际业务需求设定合理的缓存过期时间。当有数据更新或删除操作时，及时使对应的缓存数据失效，从而保证下次查询时能够从数据库获取最新的数据。
4. 消息队列：将数据更新操作通过消息队列发送到一个消费者，该消费者负责将数据写入MySQL和Redis。这种方式可以减小对数据库的直接写入操作，提高系统的可靠性和容错性。
5. 分布式事务：使用分布式事务管理框架，如Seata或TCC-Transaction等，将Redis和MySQL的写操作包装在一个分布式事务中。这样可以保证数据写入Redis和MySQL的原子性，从而避免了双写一致性问题。
6. 数据同步工具：使用数据同步工具或者自定义定时任务，定期将MySQL中的数据同步到Redis缓存中。这种方式可以保证Redis中的数据与MySQL保持一致，并且降低了实时性要求。

下面是以上方式的各自优缺点和使用场景：

1. 异步双写：
   - 优点：减小对数据库写操作的延迟，提高响应速度。
   - 缺点：可能会出现缓存与数据库之间的不一致。
   - 使用场景：适用于对数据一致性要求不高，但对响应速度要求较高的场景。
2. 同步双写：
   - 优点：保证缓存和数据库之间的数据一致性。
   - 缺点：增加请求的响应时间。
   - 使用场景：适用于对数据一致性要求较高，可以容忍稍微增加一些响应时间的场景。
3. 数据失效策略：
   - 优点：根据实际业务需求设定合理的缓存过期时间，保证下次查询时能够从数据库获取最新的数据。
   - 缺点：存在一定的数据读取时缓存与数据库不一致的风险。
   - 使用场景：适用于对数据实时性要求相对较低，可以容忍一定的数据不一致的场景。
4. 消息队列：
   - 优点：减小对数据库的直接写入操作，提高系统可靠性和容错性。
   - 缺点：增加了系统的复杂性，需要引入消息队列和消费者的相关组件。
   - 使用场景：适用于对数据一致性要求高，系统需要保证可靠性和容错性的场景。
5. 分布式事务：
   - 优点：保证Redis和MySQL的写操作在一个分布式事务中的原子性，解决双写一致性问题。
   - 缺点：增加了系统的复杂性，引入了分布式事务管理框架，可能会影响系统性能。
   - 使用场景：适用于对数据一致性要求严格，分布式事务支持较好的场景。
6. 数据同步工具：
   - 优点：保证Redis中的数据与MySQL保持一致。
   - 缺点：存在一定的数据同步延迟，不适合实时性要求较高的场景。
   - 使用场景：适用于对数据实时性要求相对较低，可以容忍一定的数据同步延迟的场景。



上面都是chatggpt给出的答案，仅供参考哈，所以我也分析了一下，最简单的就是同步双写，正好符合我现在的需求，不过我计划再使用消息队列的方式，两种方式实现，研究一下消息队列的方式。

#### 5.4.2 使用同步双写的方式

添加数据的时候，需要更新数据库的图表状态，那么就需要进行一次redis缓存，所以我单独写了一个方法，实时获取数据库的信息进行更新缓存。

```java
/**
 * 更新redis的分页缓存的第一页
 */
@Override
public void updateRedisPageCache(){
    int pageNum = 1;
    int pageSize = 4;
    // 获取当前登录用户的Id
    Long userId = SecurityUtils.getUserId();
    QueryWrapper<BiChart> wrapper = new QueryWrapper<>();
    wrapper.eq("user_id", userId);
    // 执行查询数据库操作
    Page<BiChart> result = chartService.page(new Page<>(pageNum, pageSize), wrapper);

    // 每个用户的图表都是不一样的，所以拼接userId，就是唯一的
    String pageUser = CACHE_USER_PAGE + userId;
    // 根据需要查询的当前页码和每页大小拼接
    String userPageArg = CACHE_PAGE_ARG + pageNum + ":" + pageSize;
    // 删除缓存
    Boolean delete = redisTemplate.delete(pageUser);
    log.debug("分页缓存删除结果为："+delete);
    // 存储结果到缓存中
    redisTemplate.opsForHash().put(pageUser, userPageArg, result);
    redisTemplate.expire(pageUser, CACHE_MINUTES_FIVE, TimeUnit.MINUTES);
}
```

在需要更新数据的地方下面都添加一个方法，就实现了，有点偷懒和简单，完美~

更新数据主要是添加新的图表的时候，在impl里有一个，在消费者里有更新图表状态的地方也有，都需要进行更新缓存

#### 5.4.3 使用消息队列解决双写一致性问题

其实本质跟同步更新是差不多，主要是实现了异步方式，适用于对数据一致性要求高，系统需要保证可靠性和容错性的场景。

首先创建队列

```java
/**
 * 用于创建测试程序用到的交换机和队列（只用在程序启动前执行一次）
 */
public class BiInitMain {

    public static void main(String[] args) {
        try {
            ConnectionFactory factory = new ConnectionFactory();
            factory.setHost("localhost");

            // 创建与 RabbitMQ 服务器的连接
            Connection connection = factory.newConnection();

            // 创建一个通道
            Channel channel = connection.createChannel();

            String BI_EXCHANGE_NAME = BiMqConstant.BI_EXCHANGE_NAME;
            String BI_EXCHANGE_DEAD = BiMqConstant.BI_EXCHANGE_DEAD;

            // 声明一个直连交换机，一个死信交换机（其实就是普通交换机）
            channel.exchangeDeclare(BI_EXCHANGE_NAME, "direct");
            channel.exchangeDeclare(BI_EXCHANGE_DEAD, "direct");

            // 创建一个队列，用做智能分析，分配一个队列名称
            String queueName = BiMqConstant.BI_QUEUE_NAME;
            // 创建一个队列，用做实现mysql和redis双写一致，分配一个队列名称
            String queueRedisAndMysqlName = BiMqConstant.BI_Redis_Mysql;

            // 通过设置 x-message-ttl 参数来指定消息的过期时间
            Map<String, Object> queueArgs = new HashMap<>();
            queueArgs.put("x-message-ttl", 60000); // 过期时间为 60 秒
            // 声明队列绑定
            // 参数解释：queueDeclare(String queue, boolean durable, boolean exclusive, boolean autoDelete, Map<String, Object> arguments)
            // durable: 持久化队列（重启后依然存在）
            // exclusive: 排他性队列（仅限此连接可见，连接关闭后队列删除）
            // autoDelete: 自动删除队列（无消费者时自动删除）
            channel.queueDeclare(queueName, true, false, false, queueArgs);
            channel.queueDeclare(queueRedisAndMysqlName,true,false,false,null);

            // 设置队列的高级参数，与死信交换机做交互
            String deadLetterRoutingKey = ""; // 空字符串，表示所有过期消息都会路由到死信交换机
            Map<String, Object> deadArgs = new HashMap<>();
            // 指定死信交换机的名称为 BI_EXCHANGE_DEAD。这个参数告诉 RabbitMQ 在某个消息过期时将其路由到死信交换机。
            deadArgs.put("x-dead-letter-exchange", BI_EXCHANGE_DEAD);
            // 指定死信消息的路由键。在这个示例中，deadLetterRoutingKey是一个空字符串，表示无论过期消息的原始路由键是什么，它们都会被路由到死信交换机。
            deadArgs.put("x-dead-letter-routing-key", deadLetterRoutingKey);

            // 将队列与交换机进行绑定
            // queueBind()方法的参数解析如下：
            // queueName：要绑定的队列的名称。
            // exchangeName：要绑定的交换机的名称。
            // routingKey：绑定的路由键。当生产者发送消息时，会指定一个路由键，交换机根据这个路由键将消息路由到对应的队列。
            // arguments：可选参数，用于传递额外的参数。在你的代码中，你使用了deadArgs参数来指定私信队列的相关配置。
            channel.queueBind(queueName, BI_EXCHANGE_NAME, BiMqConstant.BI_ROUTING_KEY,deadArgs);
            channel.queueBind(queueRedisAndMysqlName, BI_EXCHANGE_NAME, BiMqConstant.BI_REDISSQL_KEY, deadArgs);

            // 创建一个死信队列
            String queueDeadName = BiMqConstant.BI_QUEUE_DEAD;
            // 声明死信队列，并将其绑定到死信交换机。
            channel.queueDeclare(queueDeadName,true,false,false,null);
            channel.queueBind(queueDeadName,BI_EXCHANGE_DEAD,"");

        } catch (Exception e) {
            // 处理异常情况
        }
    }
}
```

然后在生产者里面多写一个方法：

```java
/**
 * 发送消息，异步进行redis和mysql双写一致处理开启
 * @param message
 */
public void sendMysqlAndRedis(String message) {
    rabbitTemplate.convertAndSend(BiMqConstant.BI_EXCHANGE_NAME, BiMqConstant.BI_REDISSQL_KEY, message);
}
```

然后在消费者里面写好双写一致的方法：

```java
@SneakyThrows
@RabbitListener(queues = {BiMqConstant.BI_QUEUE_NAME}, ackMode = "MANUAL")
public void receiveMysqlAndRedis(String message, Channel channel, @Header(AmqpHeaders.DELIVERY_TAG) long deliveryTag) {
    log.info("bi start receiveMysqlAndeRedis message = {}", message);

    if (StringUtils.isBlank(message)) {
        // 如果失败，消息拒绝
        channel.basicNack(deliveryTag, false, false);
        throw new BusinessException(ErrorCode.SYSTEM_ERROR, "消息为空");
    }

    int pageNum = 1;
    int pageSize = 4;
    // 获取当前登录用户的Id
    Long userId = SecurityUtils.getUserId();
    QueryWrapper<BiChart> wrapper = new QueryWrapper<>();
    wrapper.eq("user_id", userId);
    // 执行查询数据库操作
    Page<BiChart> result = chartService.page(new Page<>(pageNum, pageSize), wrapper);

    // 每个用户的图表都是不一样的，所以拼接userId，就是唯一的
    String pageUser = CACHE_USER_PAGE + userId;
    // 根据需要查询的当前页码和每页大小拼接
    String userPageArg = CACHE_PAGE_ARG + pageNum + ":" + pageSize;
    // 删除缓存
    Boolean delete = redisTemplate.delete(pageUser);
    log.debug("分页缓存删除结果为："+delete);
    // 存储结果到缓存中
    redisTemplate.opsForHash().put(pageUser, userPageArg, result);
    redisTemplate.expire(pageUser, CACHE_MINUTES_FIVE, TimeUnit.MINUTES);
    // 更新缓存
    chartService.updateRedisPageCache();
    // 消息确认
    channel.basicAck(deliveryTag, false);
    log.info("bi receiveMysqlAndeRedis message end");
}
```

然后，在需要更新数据的地方进行调用，发起任务

```java
// 指定程序监听的消息队列和确认机制
@SneakyThrows
@RabbitListener(queues = {BiMqConstant.BI_QUEUE_NAME}, ackMode = "MANUAL")
public void receiveMessage(String message, Channel channel, @Header(AmqpHeaders.DELIVERY_TAG) long deliveryTag) {
    log.info("bi start receiveMessage message = {}", message);

    if (StringUtils.isBlank(message)) {
        // 如果失败，消息拒绝
        channel.basicNack(deliveryTag, false, false);
        throw new BusinessException(ErrorCode.SYSTEM_ERROR, "消息为空");
    }

    long chartId = Long.parseLong(message);
    BiChart chart = chartService.getById(chartId);
    if (chart == null) {
        channel.basicNack(deliveryTag, false, false);
        throw new BusinessException(ErrorCode.NOT_FOUND_ERROR, "图表为空");
    }

    // 先修改图表任务状态为 “执行中”。
    // 等执行成功后，修改为 “已完成”、保存执行结果；
    // 执行失败后，状态修改为 “失败”，记录任务失败信息。
    BiChart updateChart = new BiChart();
    updateChart.setId(chart.getId());
    // 修改图表任务状态为 “执行中”
    updateChart.setChartStatus(ChartStatus.RUNNING.getValue());
    boolean b = chartService.updateById(updateChart);
    if (!b) {
        channel.basicNack(deliveryTag, false, false);
        chartService.handleChartUpdateError(chart.getId(), "更新图表执行中状态失败");
        return;
    }
    // 更新缓存
    // chartService.updateRedisPageCache();
    biMessageProducer.sendMysqlAndRedis("进行双写一致处理");
    // 调用 AI
    String chartResult = aiManager.doChat(CommonConstant.BI_MODEL_ID, chartService.buildUserInput(chart));
    // 处理ai生成的数据
    OrganizeAiDataVo organizeAiData = chartService.getOrganizeAiData(chartResult);
    String genChart = organizeAiData.getGenChart();
    String genResult = organizeAiData.getGenResult();

    BiChart updateChartResult = new BiChart();
    updateChartResult.setId(chart.getId());
    updateChartResult.setGenChart(genChart);
    updateChartResult.setGenResult(genResult);
    // 等执行成功后，修改为 “已完成”、保存执行结果；
    updateChartResult.setChartStatus(ChartStatus.SUCCEED.getValue());
    boolean updateResult = chartService.updateById(updateChartResult);
    if (!updateResult) {
        channel.basicNack(deliveryTag, false, false);
        // 执行失败后，状态修改为 “失败”，记录任务失败信息。
        chartService.handleChartUpdateError(chart.getId(), "更新图表成功状态失败");
    }
    // 更新缓存
    // chartService.updateRedisPageCache();
    biMessageProducer.sendMysqlAndRedis("进行双写一致处理");
    // 消息确认
    channel.basicAck(deliveryTag, false);
    log.info("bi receiveMessage message end");
}
```

最后就实现啦~

我猛点，直接干爆了。。。

![image-20230725154817548](https://paperfly-blog.oss-cn-shenzhen.aliyuncs.com/blog/image-20230725154817548.png)

我怀疑我做的有问题，好像并没有删除redis缓存，但是这个消息的确是接受到了，我也后续没有去看哪里除了问题，摆烂啦~

大概的思路应该是没问题的吧，啊哈

# 本项目TODO 

1. 注册，登陆 ，管理员新增 / 删除用户、管理员 / 用户修改信息   √ 
2. 在图表管理界面展示图表生成的时间信息  √ 
3. 我的图表页面增加一个刷新、定时自动刷新的按钮，保证获取到图表的最新状态（前端轮询）
4. 使用redis存储分页数据，通过缓存空字符串解决缓存穿透问题，并且使用消息队列解决双写一致性问题。
5. 给任务增加一个超时时间1分钟，超时自动标记为失败（超时控制），并且放置死信队列中可以进行二次操作



未完成：

1. 用户查看原始数据 
2. 支持跳转图表编辑界面，去编辑页 
4. 用户可以修改生成失败的图表信息，点击重新生成 
5. 反向压力：https://zhuanlan.zhihu.com/p/404993753，通过调用的服务状态来选择当前系统的策略（比如根据 AI 服务的当前任务队列数来控制咱们系统的核心线程数)，从而最大化利用系统资源。
6. 任务执行成功或失败，给用户发送实时消息通知（实时：websocket、server side event) 