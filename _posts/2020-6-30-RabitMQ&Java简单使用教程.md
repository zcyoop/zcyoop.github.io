---
layout: post
title: 'RabitMQ&Java简单使用教程'
date: 2020-6-30
tags: rabbitmq java
categories: [框架,rabbitmq]
---

## RabbitMQ&Java使用说明

### RabbitMQ简介

**RabbitMQ**是实现了高级消息队列协议（AMQP）的开源消息代理软件（亦称面向消息的中间件）。RabbitMQ服务器是用[Erlang](https://baike.baidu.com/item/Erlang)语言编写的，而群集和故障转移是构建在[开放电信平台](https://baike.baidu.com/item/开放电信平台)框架上的。所有主要的编程语言均有与代理接口通讯的客户端[库](https://baike.baidu.com/item/库)。

### RabbitMQ安装

docker一键安装

```bash
# 拉去镜像（后缀为management表示为带图形化管理界面的版本）
docker pull docker.io/rabbitmq:3.8-management
# 启动镜像
docker run -d --name rabbitmq3.7.7 -p 5672:5672 -p 15672:15672 -v `pwd`/data:/var/lib/rabbitmq --hostname myRabbit -e RABBITMQ_DEFAULT_VHOST=my_vhost  -e RABBITMQ_DEFAULT_USER=admin -e RABBITMQ_DEFAULT_PASS=admin [ent_id]
# -d 后台运行容器；
# --name 指定容器名；
# -p 指定服务运行的端口（5672：应用访问端口；15672：控制台Web端口号）；
# -v 映射目录或文件；
# --hostname  主机名（RabbitMQ的一个重要注意事项是它根据所谓的 “节点名称” 存储数据，默认为主机名）；
# -e 指定环境变量；（RABBITMQ_DEFAULT_VHOST：默认虚拟机名；RABBITMQ_DEFAULT_USER：默认的用户名；RABBITMQ_DEFAULT_PASS：默认用户名的密码）
```



### RabbitMQ中的五种队列

- Simplest Queue
- Work Queue
- Publish/Subscibe
- Routing
- Topics

导入依赖

```xml
<dependency>
    <groupId>com.rabbitmq</groupId>
    <artifactId>amqp-client</artifactId>
    <version>3.4.1</version>
</dependency>
```



#### SimplestQueue（简单队列）

![](https://blog-1253533258.cos.ap-shanghai.myqcloud.com/2020-6-10/rabbitmq_1.png)

- p(producing):生产者，用于生产消息并推送到队列中
- 红色：消息队列，用于缓存生产者推送的消息，消费者可以从中取出消息
- c(Consuming):消费者，读取队列中的消息

**代码**

**工具方法码**

```java
//用于返回一个连接
public static Connection getConnection() throws Exception {
        //定义连接工厂
        ConnectionFactory factory = new ConnectionFactory();
        //设置服务地址
        factory.setHost("localhost");
        //端口
        factory.setPort(5672);
        //设置账号信息，用户名、密码、vhost
        factory.setVirtualHost("testhost");
        factory.setUsername("admin");
        factory.setPassword("admin");
        // 通过工程获取连接
        Connection connection = factory.newConnection();
        return connection;
   }

```

**生产者**

```java
// 获取到连接以及mq通道
Connection connection = ConnectionUtil.getConnection();
// 从连接中创建通道
Channel channel = connection.createChannel();

// 声明（创建）队列
channel.queueDeclare(QUEUE_NAME, false, false, false, null);

// 消息内容
String message = "Hello World!";
channel.basicPublish("", QUEUE_NAME, null, message.getBytes());
System.out.println(" [x] Sent '" + message + "'");
//关闭通道和连接
channel.close();
connection.close();
```

**消费者**

```java
// 获取到连接以及mq通道
Connection connection = ConnectionUtil.getConnection();
// 从连接中创建通道
Channel channel = connection.createChannel();
// 声明队列
channel.queueDeclare(QUEUE_NAME, false, false, false, null);

// 定义队列的消费者
QueueingConsumer consumer = new QueueingConsumer(channel);

// 监听队列
channel.basicConsume(QUEUE_NAME, true, consumer);

// 获取消息
while (true) {
    QueueingConsumer.Delivery delivery = consumer.nextDelivery();
    String message = new String(delivery.getBody());
    System.out.println(" [x] Received '" + message + "'");
}

```



#### Work Queue

![](https://blog-1253533258.cos.ap-shanghai.myqcloud.com/2020-6-10/rabbitmq_2.png)

- 一个生产者、两个消费者
- 一条消息只能被一个消费者读取

**生产者**

```java
String QUEUE_NAME = "test_queue_work"; 

// 获取到连接以及mq通道
Connection connection = ConnectionUtil.getConnection();
Channel channel = connection.createChannel();

// 声明队列
channel.queueDeclare(QUEUE_NAME, false, false, false, null);

for (int i = 0; i < 100; i++) {
    // 消息内容
    String message = "" + i;
    channel.basicPublish("", QUEUE_NAME, null, message.getBytes());
    System.out.println(" [x] Sent '" + message + "'");

    Thread.sleep(i * 10);
}

channel.close();
connection.close();
```

**消费者X2**

```java
String QUEUE_NAME = "test_queue_work";
// 获取到连接以及mq通道
Connection connection = ConnectionUtil.getConnection();
Channel channel = connection.createChannel();

// 声明队列
channel.queueDeclare(QUEUE_NAME, false, false, false, null);

// 同一时刻服务器只会发一条消息给消费者
//channel.basicQos(1);

// 定义队列的消费者
QueueingConsumer consumer = new QueueingConsumer(channel);
// 监听队列，false表示手动返回完成状态，true表示自动
channel.basicConsume(QUEUE_NAME, true, consumer);

// 获取消息
while (true) {
    QueueingConsumer.Delivery delivery = consumer.nextDelivery();
    String message = new String(delivery.getBody());
    System.out.println(" [y] Received '" + message + "'");
    // 返回确认状态，注释掉表示使用自动确认模式
    //channel.basicAck(delivery.getEnvelope().getDeliveryTag(), false);
}
```

两种分发模式

- 轮询分发 ：使用任务队列的优点之一就是可以轻易的并行工作。如果我们积压了好多工作，我们可以通过增加工作者（消费者）来解决这一问题，使得系统的伸缩性更加容易。在默认情况下，RabbitMQ将逐个发送消息到在序列中的下一个消费者(而不考虑每个任务的时长等等，且是提前一次性分配，并非一个一个分配)。平均每个消费者获得相同数量的消息。这种方式分发消息机制称为Round-Robin（轮询）。

- 公平分发 ：虽然上面的分配法方式也还行，但是有个问题就是：比如：现在有2个消费者，所有的奇数的消息都是繁忙的，而偶数则是轻松的。按照轮询的方式，奇数的任务交给了第一个消费者，所以一直在忙个不停。偶数的任务交给另一个消费者，则立即完成任务，然后闲得不行。而RabbitMQ则是不了解这些的。这是因为当消息进入队列，RabbitMQ就会分派消息。它不看消费者为应答的数目，只是盲目的将消息发给轮询指定的消费者。

默认情况下是使用的轮询分发模式。将上述代码注释移除，并将`channel.basicConsume(QUEUE_NAME, false, consumer);`设置为false，则会采用公平分发

#### Publish/Subscibe（订阅模式）

![](https://blog-1253533258.cos.ap-shanghai.myqcloud.com/2020-6-10/rabbitmq_4.png)

![](https://blog-1253533258.cos.ap-shanghai.myqcloud.com/2020-6-10/rabbitmq_7.png)

- 一个生产者，多个消费者
- 每个消费者都有自己的队列
- 生产者没有将消息直接发送到队列，而是发送到了交换机
- 每个队列都要绑定到交换机
- 生产者发送的消息，经过交换机到达队列，实现一个消息被多个消费者获取的目的

PS：一个消费者队列可以有多个消费者实例，只有其中一个消费者实例会消费

**生产者**

```java
// 交换机名称
String EXCHANGE_NAME = "test_exchange_fanout";
// 获取到连接以及mq通道
Connection connection = ConnectionUtil.getConnection();
Channel channel = connection.createChannel();

// 声明exchange
channel.exchangeDeclare(EXCHANGE_NAME, "fanout");

// 消息内容
String message = "Hello World!";
channel.basicPublish(EXCHANGE_NAME, "", null, message.getBytes());
System.out.println(" [x] Sent '" + message + "'");

channel.close();
connection.close();
```

PS:消息发送到没有队列绑定的交换机时，消息将丢失，因为，交换机没有存储消息的能力，消息只能存在在队列中。

**消费者**

```java
//另一个消费则需要将队列名称换成另外一个例如test_queue_work2，其他代码相同
String QUEUE_NAME = "test_queue_work1";
String EXCHANGE_NAME = "test_exchange_fanout";

// 获取到连接以及mq通道
Connection connection = ConnectionUtil.getConnection();
Channel channel = connection.createChannel();

// 声明队列
channel.queueDeclare(QUEUE_NAME, false, false, false, null);

// 绑定队列到交换机
channel.queueBind(QUEUE_NAME, EXCHANGE_NAME, "");

// 同一时刻服务器只会发一条消息给消费者
channel.basicQos(1);

// 定义队列的消费者
QueueingConsumer consumer = new QueueingConsumer(channel);
// 监听队列，手动返回完成
channel.basicConsume(QUEUE_NAME, false, consumer);

// 获取消息
while (true) {
    QueueingConsumer.Delivery delivery = consumer.nextDelivery();
    String message = new String(delivery.getBody());
    System.out.println(" [Recv] Received '" + message + "'");
    Thread.sleep(10);

    channel.basicAck(delivery.getEnvelope().getDeliveryTag(), false);
}
```

#### Routing（路由模式）

![](https://blog-1253533258.cos.ap-shanghai.myqcloud.com/2020-6-10/rabbitmq_5.png)

> 在`Publish/Subscibe`模式中，所有的消息均会发送到所有的消费者，但是目前有这样一个场景，所有的日志记录必须发送到消费者A，用于记录消息，但是只有错误的日志需要发送到消费者B，这是就需`Exchange`有过滤功能
>
> 在`Routing`模式下，就可以实现这个功能

生产者

```java
Connection connection = ConnectUtils.getConnection();
Channel channel = connection.createChannel();

//声明Exchange
channel.exchangeDeclare(EXCHANGE_NAME,"direct");

//分别发送两条消息到"delete"、"insert"渠道
channel.basicPublish(EXCHANGE_NAME,"delete",null,"删除商品".getBytes());
channel.basicPublish(EXCHANGE_NAME,"insert",null,"插入商品".getBytes());

channel.close();
connection.close();
```

消费者A

```java
Connection connection = ConnectUtils.getConnection();
Channel channel = connection.createChannel();

channel.queueDeclare(QUEUE_NAME1,false,false,false,null);

//绑定到交换机，接受"insert"、"delete"两个渠道的消息，也就是最终结果会受到两条消息
channel.queueBind(QUEUE_NAME1,EXCHANGE_NAME,"insert");
channel.queueBind(QUEUE_NAME1,EXCHANGE_NAME,"delete");

channel.basicQos(1);

QueueingConsumer consumer = new QueueingConsumer(channel);

// 监听队列，手动返回完成
channel.basicConsume(QUEUE_NAME1, false, consumer);
// 获取消息
while (true) {
    QueueingConsumer.Delivery delivery = consumer.nextDelivery();
    String message = new String(delivery.getBody());
    System.out.println(" [Recv1] Received '" + message + "'");
    Thread.sleep(10);

    channel.basicAck(delivery.getEnvelope().getDeliveryTag(), false);
}
```

消费者B

```java
Connection connection = ConnectUtils.getConnection();
Channel channel = connection.createChannel();

channel.queueDeclare(QUEUE_NAME2,false,false,false,null);

//绑定到交换机，只绑定了"delete"渠道，也就是只会受到一条消息
channel.queueBind(QUEUE_NAME2,EXCHANGE_NAME,"delete");

channel.basicQos(1);

QueueingConsumer consumer = new QueueingConsumer(channel);

// 监听队列，手动返回完成
channel.basicConsume(QUEUE_NAME2, false, consumer);
// 获取消息
while (true) {
    QueueingConsumer.Delivery delivery = consumer.nextDelivery();
    String message = new String(delivery.getBody());
    System.out.println(" [Recv2] Received '" + message + "'");
    Thread.sleep(10);

    channel.basicAck(delivery.getEnvelope().getDeliveryTag(), false);
}
```

#### Topics（主题模式）

![](https://blog-1253533258.cos.ap-shanghai.myqcloud.com/2020-6-10/rabbitmq_6.png)

> 主题模式是路由模式的一个升级，在过滤条件上更加灵活
>
> 主题模式是将路由键和某个模式进行匹配。此时队列需要绑定一个模式上。`#`匹配一个或多个词，`*`匹配不多不少一个词。因此`audit.#`能够匹配到`audit.irs.corporate`，但是`audit.*`只会匹配到`audit.irs`

生产者

```java
Connection connection = ConnectUtils.getConnection();
Channel channel = connection.createChannel();

channel.exchangeDeclare(EXCHANGE_NAME, "topic");

//发送两条消息
channel.basicPublish(EXCHANGE_NAME, "routkey.1", null, "routkey消息".getBytes());
channel.basicPublish(EXCHANGE_NAME, "common.1", null, "common消息".getBytes());

channel.close();
connection.close();
```

消费者A

```java
Connection connection = ConnectUtils.getConnection();
Channel channel = connection.createChannel();

channel.queueDeclare(QUEUE_NAME1, false, false, false, null);

//绑定到交换机
channel.queueBind(QUEUE_NAME1, EXCHANGE_NAME, "routkey.#");
QueueingConsumer consumer = new QueueingConsumer(channel);
// 监听队列
channel.basicConsume(QUEUE_NAME1, true, consumer);
// 获取消息
while (true) {
    QueueingConsumer.Delivery delivery = consumer.nextDelivery();
    String message = new String(delivery.getBody());
    System.out.println(" [Recv1] Received '" + message + "'");
}
```

消费者B

```java
Connection connection = ConnectUtils.getConnection();
Channel channel = connection.createChannel();

channel.queueDeclare(QUEUE_NAME2, false, false, false, null);

//绑定到交换机
channel.queueBind(QUEUE_NAME2, EXCHANGE_NAME, "#.#");

QueueingConsumer consumer = new QueueingConsumer(channel);
channel.basicConsume(QUEUE_NAME2, true, consumer);
// 获取消息
while (true) {
    QueueingConsumer.Delivery delivery = consumer.nextDelivery();
    String message = new String(delivery.getBody());
    System.out.println(" [Recv2] Received '" + message + "'");
}
```