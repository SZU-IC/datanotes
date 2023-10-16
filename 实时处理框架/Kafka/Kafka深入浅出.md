# Kafka深入浅出

## 一、消息系统

### 1、解耦

允许你独立的扩展或修改两边的处理过程，只要确保他们遵守同样的接口约束。

### 2、冗余

消息队列把数据进行持久化直到他们已经被完全处理，通过这一方式规避了数据丢失风险。许多消息队

列所采用的"插入-获取-删除"范式中，在把一个消息从队列中删除之前，需要你的处理系统明确的指出

该消息已经被处理完毕，从而确保你的数据被安全的保存直到你使用完毕。

### 3、扩展性

因为消息队列解耦了你的处理过程，所以增大消息入队和处理的频率是很容易的，只要另外增加处理过

程即可。

### 4、灵活性 & 峰值处理能力

在访问量剧增的情况下，应用仍然需要继续发挥作用，但是这样的突发流量并不常见。如果为以能处理

这类峰值访问为标准来投入资源随时待命无疑是巨大的浪费。使用消息队列能够使关键组件顶住突发的

访问压力，而不会因为突发的超负荷的请求而完全崩溃。

### 5、可恢复性

系统的一部分组件失效时，不会影响到整个系统。消息队列降低了进程间的耦合度，所以即使一个处理

消息的进程挂掉，加入队列中的消息仍然可以在系统恢复后被处理。

### 6、顺序保证

在大多使用场景下，数据处理的顺序都很重要。大部分消息队列本来就是排序的，并且能保证数据会按

照特定的顺序来处理。（Kafka 保证一个 Partition 内的消息的有序性）

### 7、缓冲

有助于控制和优化数据流经过系统的速度，解决生产消息和消费消息的处理速度不一致的情况。

### 8、异步通信

很多时候，用户不想也不需要立即处理消息。消息队列提供了异步处理机制，允许用户把一个消息放入

队列，但并不立即处理它。想向队列中放入多少消息就放多少，然后在需要的时候再去处理它们。

## 二、Kafka核心概念

![image-20221215105157439](.\kafka架构.png)

Apache Kafka是分布式发布-订阅消息系统。它最初由LinkedIn公司开发，之后成为Apache项目的一部

分。Kafka是一种快速、可扩展的、设计内在就是分布式的，分区的和可复制的提交日志服务。

### 1、Producer

消息的生产者（client），发布消息到kafka集群的终端或服务。

## 2、Broker

Kafka集群中包含的服务器（JVM进程？）

### 3、Topic

每条发布到Kafka集群的消息属于的类别，即Kafka是面向topic的。（类似于RDD）

### 4、Partition

partition是物理上的概念，每个topic包含一个或多个partition。Kafka分配的单位是partition

### 5、Consumer

从Kafka集群中消费消息的终端或服务。

### 6、Consumer Group

 high-level consumer API 中，每个 consumer 都属于一个 consumer group，每条消息只能被

consumer group 中的一个 Consumer 消费，但可以被多个 consumer group 消费。（消费方式：轮询方式）

### 7、Replica

partition的副本，保障partition高可用

### 8、leader

replica 中的一个角色， producer 和 consumer 只跟 leader 交互。

### 9、follower

replica 中的一个角色，从 leader 中复制数据。

### 10、Controller

1. Kafka集群中某个broker宕机之后，负责感知他的宕机，以及负责Leader partition的选举
2. Kafka集群重新加入新的机器，负责数据的迁移和负载均衡
3. 管理Kafka集群上的元数据（比较大，在zookeeper中）
4. 删除topic，并删除topic中的partition
5. 监听broker

### 11、Zookeeper

1. Kafka通过zookeeper来存储集群的meta信息。

2. 一旦controller所在broker宕机了，此时临时节点消失，集群里其他broker会一直监听这个临时节

   点，发现临时节点消失了，就争抢再次创建临时节点，保证有一台新的broker会成为controller角色。

### 12、问题

数据流程变慢？数据丢失怎么办？消费者跟不上生产者怎么办？宕机怎么办？

## 三、Kafka高性能高可用的原理

### 1、顺序写（appendonly）

Kafka仅仅是追加数据到文件末尾，磁盘顺序写，性能极高，在磁盘转数较高的情况下，写入速度也可以接近内存。（寻道时间、寻址时间变短）

### 2、零拷贝（log sendfile）

![image-20221215112357352](.\读取数据-常规.png)

![image-20221215112509923](.\读取数据-零拷贝.png)

### 3、服务端分段存储

Topic进行分区存储到各个服务器，分区中包含三种文件：index、log、timeindex

![image-20221215112855761](.\分段存储文件.png)

文件名前面的数字，代表了这个日志段文件包含的起始offset。

Kafka broker有一个参数，log.segment.bytes，限定了每个日志段文件的大小，最大就是1GB，一个日志段文件满了，就自动开一个新的日志段文件来写入，避免单个文件过大，影响文件的读写性能，这个过程叫做log rolling，正在被写入的那个日志段文件，叫做active log segment。

### 4、服务端冗余副本

0.8版本之前没有，可以实现负载均衡，利用多个服务器的读写性能（前提是leader partition要均衡分配到各个机器）。

leader partition：承载读写数据的压力、维护ISR副本列表（保证所有的副本写入成功）

follower partition：从leader partition同步数据

### 5、稀疏索引、二分查找

index文件：

- offset：相对偏移量（log文件名前面的数字）
- position：物理位置

## 四、资源规划

### 1、需求分析（举例）

1. 请求数：10亿
2. 二八法则
   1. 晚上12：00 --- 早上 8：00                  20%数据
   2. 其余时间                                                80%数据
   3. 高峰期（二八法则）： 16 * 20% = 3小时     80% * 80% * 10亿 = 6亿
   4. qps（每秒）： 6亿 / 3小时 = 5.5万条
3. 副本数： 2
4. 保留周期： 3天（根据自己的业务评判）

### 2、资源

1. 评估： 按照高峰期的 4 倍进行评估
2. 磁盘：
3. CPU（100多个线程）
   1. 接收请求线程
   2. 处理请求线程
   3. 定时清理线程

## 五、Kafka Producer、Consumer消息发送原理

### 1、Producer 发送步骤

![image-20221220162732792](.\Producer发送原理.png)

1. 每次发送消息都会把数据封装成一个ProducerRecord对象，里面包含发送的Topic，具体的分区，分区key，消息内容，timestamp时间戳。
2. 将对象交给序列化器，序列化成自定义协议格式的数据
3. 将数据交给partitioner分区器，对数据选择合适的分区（轮询或者hash路由）
4. 数据发送到prodcuer内部的缓冲区，producer内部的sender线程将缓冲区的数据封装成一个个batch，发送给分区的leader副本所在的broker。

~~~Java
import java.util.Properties;
import org.apache.kafka.clients.producer.Callback;
import org.apache.kafka.clients.producer.KafkaProducer;
import org.apache.kafka.clients.producer.ProducerRecord;
import org.apache.kafka.clients.producer.RecordMetadata;
public class ProducerDemo {
public static void main(String[] args) throws Exception {
Properties props = new Properties();
// 这里可以配置几台broker即可，他会自动从broker去拉取元数据进行缓存
props.put("bootstrap.servers",
"hadoop03:9092,hadoop04:9092,hadoop05:9092");
Producer核心参数
常见异常处理
// 这个就是负责把发送的key从字符串序列化为字节数组
props.put("key.serializer",
"org.apache.kafka.common.serialization.StringSerializer");
// 这个就是负责把你发送的实际的message从字符串序列化为字节数组
props.put("value.serializer",
"org.apache.kafka.common.serialization.StringSerializer");
props.put("acks", "-1");
props.put("retries", 3);
props.put("batch.size", 323840);
props.put("linger.ms", 10);
props.put("buffer.memory", 33554432);
props.put("max.block.ms", 3000);
// 创建一个Producer实例：线程资源，跟各个broker建立socket连接资源
KafkaProducer<String, String> producer = new KafkaProducer<String,
String>(props);
ProducerRecord<String, String> record = new ProducerRecord<>(
"test-topic", "test-key", "test-value");
```
// 这是异步发送的模式
producer.send(record, new Callback() {
@Override
public void onCompletion(RecordMetadata metadata, Exception exception) {
if(exception == null) {
// 消息发送成功
System.out.println("消息发送成功");
} else {
// 消息发送失败，需要重新发送
}
}
});
Thread.sleep(10 * 1000);
// 这是同步发送的模式
// producer.send(record).get();
// 你要一直等待人家后续一系列的步骤都做完，发送消息之后
// 有了消息的回应返回给你，你这个方法才会退出来
producer.close();
	}
}
~~~

### 2、Producer参数

#### 1、提升吞吐量

1. buffer.memory：设置发送消息的缓冲区，默认值是33554432，即32M。
   1. 发送消息的速度大于写入消息的速度，就会导致缓冲区写满，生产消息就会阻塞。
2. compression.type:默认是none，不压缩，可以使用lz4压缩，压缩后减少数据量，提升吞吐，但是会加大producer的cpu开销。
3. batch.size：设置每个batch的大小，如果batch太小，会导致频繁网络请求，吞吐量下降；如果batch太大，会导致一条消息需要等待很久才能被发送出去，而且会让内存缓冲区有很大压力，过多的
   1. 默认值：16384，16KB
4. linger.ms:默认值为0，意思就是消息一来就立刻发送出去。一般设置100ms，就是100ms内这个消息满16kb，自然会发送出去，100ms内没满也会发送出去。

#### 2、请求超时

1. request.timeout.ms:这个就是说发送一个请求出去之后，他有一个超时的时间限制，默认是30秒，

   如果30秒都收不到响应，那么就会认为异常，会抛出一个TimeoutException来让我们进行处理

2. max.request.size:这个参数用来控制发送出去的消息的大小，默认是1048576字节，也就1mb，这个

一般太小了，很多消息可能都会超过1mb的大小，所以需要自己优化调整，把他设置更大一些（企业一般设置

成10M)

#### 3、ACK参数

1. acks=0：producer不管broker的消息到底成功了没有，发送一条消息出去，立马就可以发送下一条消息，吞吐最高。
2. acks=1：只要leader写入成功了，就认为消息成功了，如果刚写入leader，leader就挂了，此时数据必然丢失。
3. acks=-1：leader写入成功后，需要等待其他的ISR中的副本都写入成功，才可以返回响应说这条消息写入成功了，这是就会收到一条回调的通知。
4. min.insync.replicas=2，ISR里必须有2个副本，一个leader和一个follower
5. retries=Integer.MAX_VALUE，无线重试
6. 要想数据不丢失，必须保证3 4 5

#### 4、重试乱序（分区内的）

消息重试是可能导致消息的乱序的，因为可能排在你后面的消息都发送出去了，你现在收到回调失败了才再重试，此时消息就可能乱序，可以使用"max.in.flight.requests.per.connection"参数设置为1，这样可以保证producer同一时间只能发送一条消息。

#### 5、生产者异常

1)LeaderNotAvailableException：
   这个就是如果某台机器挂了，此时leader副本不可用，会导致你写入失败.
   要等待其他follower副本切换为leader副本之后，才能继续写入，此时可以重试发送即可
   如果说你平时重启kafka的broker进程，肯定会导致leader切换，一定会导致你写入报错，是LeaderNotAvailableException。

2)NotControllerException：这个也是同理，如果说Controller所在Broker挂了，
   那么此时会有问题，需要等待Controller重新选举，此时也是一样就是重试即可

3)NetworkException：网络异常  timeout

4）重试机制参数

```java
props.put("retries",10);
props.put("retry.backoff.ms",300);
```

其他异常：备用链路处理：redis，不重要的数据删除

#### 6、生产者重试带来的问题

1、消息会重复

有的时候一些leader切换等问题，需要进行重试，设置retries即可，消息重试会导致重复发送的问题，比如网络抖动导致没有收到回调的信息，以为失败了。

2、消息乱序

消息重试是可能导致消息的乱序的，因为可能排在你后面的消息都发送出去了。

#### 7、不丢数据的方案

1. 创建主题的分区副本>=2,3,4
2. acks=-1
3. min.insync.replicas>=2

### 3、Consumer

#### 1、offset管理

1. 老版本：offset的管理是写入zk，对于高并发请求zk是不合理的，zk只是做分布式系统的协调，轻量级的元数据存储，不能负责高并发读写。
2. 新版本：offset提交到内部topic:consumer_offset ,key是 group.id+topic+分区号,value是当前offset的值，每隔一段时间，kafka内部对这个topic进行compact，也就是每个key保留最新的那条数据。consumer_offsets默认的分区为50

#### 2、Coordinator（consumer group中的leader）

1. 作用
   1. 每个consumer group都会选择一个broker作为自己的coordinator，负责监控这个消费者组的消费者的心跳，以及判断是否宕机，然后进行rebalance。
2. 选择coordinator
   1. 先对消费者组的groupid进行hash，接着consumer_offsets的分区数量取模，默认是50，可以通过offsets.topic.num.partitions来设置，找到consumer group的offset要提交到consumer_offsets哪个分区。
   2. 找到consumer_offsets的一个分区，这个分区所对应分区leader所在的broker也就是这个comsumer group的coordinator，consumer就会去维护一个Socket连接跟这个Broker进行通信。

#### 3、consumer加入并消费数据

![image-20221221113502775](.\consumer消费数据.png)

#### 4、重平衡策略

1. range策略
   1. p0~p3	consumer1
   2. p4~p7    consumer2
   3. p8~p11  consumer3
2. round-robin策略，轮询分配。
   1. consumer1:0,3,6,9
   2. consumer2:1,4,7,10
   3. consumer3:2,5,8,11
3. sticky策略
   1. 最新的一个sticky策略，就是说尽可能保证在rebalance的时候，让原本属于这个consumer的分区还是属于他们，
      然后把多余的分区再均匀分配过去，这样尽可能维持原来的分区分配的策略
4. Rebalance分代机制
   1. 在rebalance时，可能你本来要消费partition3数据，结果有些数据消费了还没提交offset，结果此时rebalance，将partition3分配给另一个consumer。
   2. rebalance时会触发一次consumer group generation分代，每次分代会加1，然后你提交上一个分代的offset是不行的。
5. 问题：
   1. range和round-robin策略在节点挂掉时导致存储位置频繁更换

```java
import java.util.Arrays;
import java.util.Properties;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import org.apache.kafka.clients.consumer.ConsumerRecord;
import org.apache.kafka.clients.consumer.ConsumerRecords;
import org.apache.kafka.clients.consumer.KafkaConsumer;
import com.alibaba.fastjson.JSONObject;
public class ConsumerDemo {
private static ExecutorService threadPool =
Executors.newFixedThreadPool(20);
public static void main(String[] args) throws Exception {
KafkaConsumer<String, String> consumer = createConsumer();
consumer.subscribe(Arrays.asList("order-topic"));
try {
while(true) {
ConsumerRecords<String, String> records =
consumer.poll(Integer.MAX_VALUE);
for(ConsumerRecord<String, String> record : records) {
JSONObject order = JSONObject.parseObject(record.value());
threadPool.submit(new CreditManageTask(order));
}
}
} catch(Exception e) {
e.printStackTrace();
consumer.close();
}
}
private static KafkaConsumer<String, String> createConsumer() {
Properties props = new Properties();
props.put("bootstrap.servers",
"hadoop03:9092,hadoop04:9092,hadoop05:9092");
props.put("group.id", "test-group");
props.put("key.deserializer",
"org.apache.kafka.common.serialization.StringDeserializer");
props.put("value.deserializer",
"org.apache.kafka.common.serialization.StringDeserializer");
props.put("heartbeat.interval.ms", 1000); // 这个尽量时间可以短一点
props.put("session.timeout.ms", 10 * 1000); // 如果说kafka broker在10秒内
感知不到一个consumer心跳
props.put("max.poll.interval.ms", 30 * 1000); // 如果30秒才去执行下一次poll
// 就会认为那个consumer挂了，此时会触发rebalance
// 如果说某个consumer挂了，kafka broker感知到了，会触发一个rebalance的操作，就是
分配他的分区
// 给其他的cosumer来消费，其他的consumer如果要感知到rebalance重新分配分区，就需要
通过心跳来感知
// 心跳的间隔一般不要太长，1000，500
props.put("fetch.max.bytes", 10485760);
props.put("max.poll.records", 500); // 如果说你的消费的吞吐量特别大，此时可以适
当提高一些
props.put("connection.max.idle.ms", -1); // 不要去回收那个socket连接
// 开启自动提交，他只会每隔一段时间去提交一次offset
// 如果你每次要重启一下consumer的话，他一定会把一些数据重新消费一遍
props.put("enable.auto.commit", "true");
// 每次自动提交offset的一个时间间隔
props.put("auto.commit.ineterval.ms", "1000");
// 每次重启都是从最早的offset开始读取，不是接着上一次
props.put("auto.offset.reset", "earliest");
KafkaConsumer<String, String> consumer = new KafkaConsumer<String,
String>(props);
return consumer;
}
static class CreditManageTask implements Runnable {
private JSONObject order;
public CreditManageTask(JSONObject order) {
this.order = order;
}
@Override
public void run() {
System.out.println("对订单进行积分的维护......" +
order.toJSONString());
// 就可以做一系列的数据库的增删改查的事务操作
}
}
}
```

#### 5、Consumer核心参数

1. heartbeat.interval.ms
   - consumer心跳时间，必须 保持心跳才能知道consumer是否故障，如果故障之后，就会通过心跳下发rebalance的指令给其他的consumer通知他们进行rebalance操作。
2. session.timeout.ms
   - kafka多长时间感知不到一个consumer就认为他故障了，默认是10秒。
3. max.poll.interval.ms
   - 如果在两次poll操作之间，超过了这个时间，那么就会认为这个consume处理能力太弱了，会被踢出消费组，分区分配给别人去消费，一遍来说结合你自己的业务处理的性能来设置就可以了.
4. fetch.max.bytes
   - 获取一条信息最大的字节数，一般建议设置大一些。
5. max.poll.records
   - 一次poll返回信息的最大条数，默认是500条。
6. connection.max.idle.ms
   - consumer跟broker的socket连接如果空闲超过一定时间，此时就会自动回收连接，但是下次消费就要重新建立socket连接，这个建议设置为-1，不用区回收。
7. auto.offset.reset
   - earliest:当各个分区下已有提交的offset，从提交的offset开始消费；无提交的offset时，从头开始消费。
   - lastest：当各个分区下已有提交的offset时，从提交的offset开始消费；无提交的offset时，从当前位置开始消费。
   - none：topic各分区都存在已提交的offset，从offset后开始消费，只要有一个分区不存在已提交的offset，则抛出异常。
8. enable.auto.commit
   - 开启自动提交
9. auto.commit.interval.ms
   - 多久提价一次偏移量

### 4、问题

```
Kafka生产者如何提升消息吞吐量?
Kafka的ack参数如何使用?
Kafka消费者组有什么作用?
说几个Kafka的Rebalance策略?
```

## 六、Controller对集群的管理

![image-20221221152946331](.\controller.png)

### 1、HW&LEO

#### 1、LEO

​	last end offset，日志末端偏移量，记录了该副本对象底层日志文件中下一条消息的位移值。

#### 2、HW

​	任何一个副本的HW值一定不大于LEO值，小于HW的offset数据是可见的。

#### 3、follower更新LEO的机制

1. follower副本所在的broker缓存里。

2. leader所在broker缓存里，也就是leader所在broker的缓存上保存了该分区所有副本的LEO。

3. follower的LEO何时更新？

   1. follower发送FETCH请求后，leader将数据返回给follower，follower开始向底层log写数据，并更新LEO值，每当新写入一条消息，其LEO值就会加1。

4. leader的LEO何时更新？

   1. 发生在leader处理follower Fetch请求时，一旦leader接收到follower发送的FETCH请求，它会从自己的log读取相应的数据，但是在给follower返回数据之前先去更新follower的LEO。
   2. leader的LEO就保存在其所在broker的缓存里，leader写log时就会自动更新其LEO值。

5. follower更新HW机制

   1. follower更新HW发生在其更新LEO之后，一旦follower向log写完数据，它就会尝试更新HW值。具体算

      法就是比较当前LEO值与FETCH响应中leader的HW值，取两者的小者作为新的HW值。这告诉我们一个

      事实：如果follower的LEO值超过了leader的HW值，那么follower HW值是不会越过leader HW值的。

6. leader更新HW机制

   1. leader更新HW的时机

      - producer向leader写消息时
      - leader处理follower的fetch请求时
      - 某副本称为leader
      - broker崩溃时导致副本被踢出ISR时

   2. leader更新HW方式

      - 选出所有满足条件的副本，比较他们的LEO，并选择最小的LEO作为HW值。

      - 满足条件：①处于ISR中

        ​					②副本LEO落后于leader LEO的时长不大于replica.lag.time.max.ms参数值（默认值是10秒）

![image-20221221175716655](.\LEO&HW.png)