# Redis（Kuangshen）

## 一、安装

挂载配置文件目录

## 二、Redis和Memcached有什么区别？

1、共同点

- 都是基于内存的数据库，一般用来当作缓存使用。
- 都有过期策略。
- 两者的性能都非常高。

2、区别

- Redis 支持的数据类型更丰富（String、Hash、List、Set、ZSet），而 Memcached 只支持最简单的 key-value 数据类型；
- Redis 支持数据的持久化，可以将内存中的数据保持在磁盘中，重启的时候可以再次加载进行使用，而 Memcached 没有持久化功能，数据全部存在内存之中，Memcached 重启或者挂掉后，数据就没了；
- Redis 原生支持集群模式，Memcached 没有原生的集群模式，需要依靠客户端来实现往集群中分片写入数据；
- Redis 支持发布订阅模型、Lua 脚本、事务等功能，而 Memcached 不支持；
- Redis一般作为MySQL的缓存

## 三、String类型

- set key value
- get key
- mset k1 v1 k2 v2 k3 v3 k4 v4
- mget k1 k2 k3 k4
- type k1
- exists key
- del key
- incr key 、decr key
- decrby key decrement 、incrby key increment

## 四、List类型

- LPUSH、RPUSH 两端添加元素
- lpop、rpop
- lrange 0 -1
- exist list 是否存在list列表
- ltrim 通过下标截取指定的长度,截取掉的就不要了；
- rpoplpush 拷贝到其他的list
- lset list index value  （容易报错，该list可能不存在）将列表中指定下标的值替换为另外一个值
- linsert 可以往某个key的前面或者后面插入某个值 

List列表实际上是一个链表，如果key不存在，创建新的链表；如果key存在，新增内容；在两边插入和改动值效率最高；

- 消息排队！消息队列

## 五、Set

set中的值不能重复

- sadd 添加值
- smemebers 查看成员
- sismember 是否存在  是返回1，否返回0
- scard 获取set集合中的个数
- srem 移除具体的元素
- srandmember 随机抽选出指定个数的元素
- spop 删除随机的key
- smove 将指定的值移动到另外一个set中

微博、B站中共同关注（并集）

数字集合类：

​	-- 差集	sdiff

​	--交集	sinter

​	--并集	sunion

## 六、Hash（map集合）

Map集合，key-map<key-value>

- hset hash key value
- hget hash key
- hmset hash key value k v k1 v1..
- hmget hash k1 k2 k3
- hgetall hash获取所有的hash值
- hdel 删除hash指定的key
- hlen 获取hash长度  
- hexists hash key 判断hash中指定的key是否存在
- hkeys hash
- hvals hash
- hincrby 
- hdecrby
- hsetnx

hash变更的数据，user name age，尤其是用户信息，经常变动的信息。hash更适合对象的存储，string更适合字符串。



## 七、ZSet有序集合

- zadd
- zrange
- zrangebyscore salary -inf +inf  从小到大排序
- zrevrange salary 0 -1 从大到小排序
- zrem 移除指定集合中的元素
- zcard 获取有序集合中的个数
- zcount 获取指定区间的成员数量，可以使用(0 (3 给区间标定是否是开闭

案例思路：带权重进行判断

排行榜应用实现，取Top N测试

## 八、geospatial 地理位置(底层实现原理其实就是zset)

- Redis GEO在Redis3.2已推出！！！
- geoadd(地球两极没法添加) 维度经度有范围限制
- geopos
- geodist 返回两个位置之间指定的距离
- georadius 以给定的精度纬度为中国心，找出某个半径的成员
- georadiusbymeber
- geohash 返回11个字符的hash字符串（二维转化为一维的字符串）
- zrange 
- zrem

我附近的人？（获得所有附近的人的地址，定位）通过半径来查询

获取附近指定数量的人count

## 九、Hyperloglog（2.8.9版本推出）

1.什么是基数？不重复的元素

Hyperloglog基数统计的算法，可以接受误差；

优点：占用内存是固定的，2^64不同的元素基数，只需要12KB。

传统方法：set保存用户id，可以统计set中元素数量作为判断标准，会保存大量的用户id。



- pfadd
- pfcount
- pfmerge key3 key1 key2

## 十、BitMaps

位存储

统计用户信息，活跃、不活跃；登录、为登录；打卡，未打卡。两个状态

- setbit
- getbit
- bitcount 统计数量 

## 十一、事务



事务的本质：一组命令的集合！一个事务中的所有命令都会被序列化，在事务执行过程中，会按照顺序执行；

一次性、顺序性、排他性！执行一系列的命令！！！

Redis事务没有隔离级别的概念！

所有的命令在事务中，并没有直接被执行！！只有发起执行命令的时候才会执行！Exec

Redis的单条命令是保证原子性的，但是事务是不保证原子性。

redis的事务：

- 开启事务（multi）
- 命令入队（...）
- 执行事务（exec）

```shell
multi
set k1 v1 
set k2 v2 
set k3 v3
exec


multi
set k1 v1 
set k2 v2 
set k3 v3
discard   #放弃事务，事务中的命令都不会被执行
```

>编译型异常（代码有问题！命令有错！）那么中子星命令时，所有的命令都不会被执行

>运行时异常，其他命令不受影响！！！

#### 悲观锁

- 认为什么时候都会出问题，无论做什么都会加锁！Synchronize啥时候都会加锁

#### 乐观锁（秒杀系统）

- 认为什么时候都不会出问题，所以不会上锁！更新数据的时候取判断一下，在此期间是否有人修改过这个数据！
- 获取version
- 更新的时候比较version

>Redis监视测试

测试多个线程修改值后失败，使用watch可以当作redis的乐观锁操作

watch key

unwatch 如果发现事务执行失败，先解锁 （自旋锁可以做到）

## 十二、Jedis

使用Java操作Jedis，Java操作Redis中间件。

1、String 操作

```java
public class TestPing {
    public static void main(String[] args) {
        Jedis jedis = new Jedis("192.168.6.102", 6379);
        System.out.println(jedis.ping());
        jedis.set("name","wither");
        jedis.set("age","18");
        jedis.set("sex","male");

        Set<String> keys = jedis.keys("*"); //获取所有的key，返回Set
        System.out.println(keys);

        String name = jedis.get("name");    //获取对应的key的value
        System.out.println(name);

        String isSet = jedis.set("job", "big data engineer");        //设置对应的值，成功返回OK;如果有key会覆盖写
        System.out.println(isSet);

        System.out.println(jedis.get("job"));

        String girlfriend = jedis.getSet("girlfriend", "niaoniao");        //如果没有设置
        System.out.println(girlfriend);
        String girlfriend1 = jedis.get("girlfriend");
        System.out.println(girlfriend1);

        System.out.println(jedis.del("age"));
        System.out.println(jedis.keys("*"));

        jedis.set("age","18");
        jedis.incr("age");
        System.out.println(jedis.get("age"));
        jedis.incrBy("age",3);
        System.out.println(jedis.get("age"));

        jedis.expire("age",10);     //过期时间
        System.out.println(jedis.exists("name"));

        List<String> strings = jedis.aclCat();      //可以查看数据类型，和数据类型的操作方法
        System.out.println(strings);

        jedis.aclDelUser("age");        //删除用户
        System.out.println(jedis.keys("*"));
        jedis.close();
    }
}

```

2、Set操作

```java
public class TestSet {
    public static void main(String[] args) {
        Jedis jedis = new Jedis("192.168.6.102", 6379);
        System.out.println(jedis.ping());

        jedis.flushAll();

        jedis.select(0);

        jedis.sadd("user","wither");

        jedis.sadd("user","18");

        byte[] key = "user".getBytes();
        byte[] value = "male".getBytes();
        jedis.sadd(key,value);

        System.out.println(jedis.smembers("user"));        //[sex, name, age]  展示也是按照String展示

        System.out.println(jedis.spop("user"));         //18 随机弹出
        System.out.println(jedis.smembers("user"));         //[18, wither, male]
        System.out.println(jedis.srem("user","male"));      //1
        System.out.println(jedis.smembers("user"));     //[18, wither]

        System.out.println(jedis.scard("user"));        //1 获取集合中的个数
        System.out.println(jedis.smembers("user"));


        System.out.println(jedis.sismember("user","wither"));       //true

        jedis.smove("user","names","wither");       //将目标set的值移到另外一个set
        System.out.println(jedis.smembers("names"));

//        System.out.println(jedis.sunion("user", "names"));      //并集[18, wither]
//        System.out.println(jedis.sdiff("user", "names"));          //差集 【】
        System.out.println(jedis.sinter("user","names"));           //交集【】

    }
}
```

3、List

```java
public class TestList {
    public static void main(String[] args) {

        Jedis jedis = new Jedis("192.168.6.102", 6379);
        jedis.flushAll();
        jedis.select(0);
        jedis.lpush("user","wither");           //从左边添加元素
        jedis.rpush("user","18");               ////从右边添加元素
        jedis.lset("user",1,"male");          //不存在就会报错，会替换指定的下标
        jedis.lpush("user","18");
        //System.out.println(jedis.rpop("user"));         //从右边弹出

        //lmove lmpop
        System.out.println(jedis.llen("user")); //查看长度
        System.out.println(jedis.ltrim("user", 0, 1));          //裁剪指定的字段
        jedis.linsert("user", ListPosition.AFTER,"18","computer");  //在 “18” 后面添加computer

        jedis.rpoplpush("user","list1");
        System.out.println(jedis.lrange("user", 0, -1));
        System.out.println(jedis.lrange("list1", 0, -1));
        jedis.close();
    }
}
```

## 十三、Redis发布订阅

```
PSUBSCRIBE PATTERN #订阅一个或多个符合给定模式的频道

PUBSUB subcommand #查看订阅与发布系统的状态

PUBLISH CHANNEL MESSAGE #将信息发送到指定的频道

PUNSUBSCRIBE CHANNEL  #订阅给定的一个或多个频道的信息

UNSUBSCRIBE CHANNLE	#退订给定的频道
```

通过 PUBLISH 命令向订阅者发送消息，redis-server 会使用给定的频道作为键，在它所维护的 channel

字典中查找记录了订阅这个频道的所有客户端的链表，遍历这个链表，将消息发布给所有订阅者。

使用场景：

- 实时消息系统
- 实时聊天（频道当作聊天室，将信息回显给所有人）
- 订阅，关注系统都是可以的（稍微复杂的场景可以使用消息中间MQ）

