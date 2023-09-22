# Spark 原理

## 一、RDDs（Resilienct Distributed Datasets）

1. RDD

   - RDD是一个只读，可分区的数据集。
   - 不需要去具象化（抽象化）。
   - 有血缘（lineage）的信息，表示从哪个数据集计算来的。
   - 用户可以根据需要（复用RDD，关联等）控制RDDs的 数据存储和分区。

   1. 总结：RDDs：数据、只读、血缘、物化（减少重复计算）、弹性

2. 弹性分布式数据集中的弹性理解

   ```
   #《大数据经典论文精读》
   ①数据存储。数据不再是存放在硬盘上的，而是可以缓存在内存中。只有当内存不足的时候，才会把它们换出到硬盘上。同时，数据的持久化，也支持硬盘、序列化后的内存存储，以及反序列化后 Java 对象的内存存储三种形式。每一种都比另一种需要占用更多的内存，但是计算速度会更快。
   ②选择把什么数据输出到硬盘上。Spark 会根据数据计算的 Lineage，来判断某一个 RDD 对于前置数据是宽依赖，还是窄依赖的。如果是宽依赖的，意味着一个节点的故障，可能会导致大量的数据要进行重新计算，乃至数据网络传输的需求。那么，它就会把数据计算的中间结果存储到硬盘上。
   ```

3. 开发原则

   - 数据输入 ---- 能省则省
   - 网络传输 ---- 能拖则拖

4. Spark程序接口

   - Scala中lazy关键字定义惰性变量，实现延迟加载。惰性变量只能是不可变变量，只有在调用惰性变量时才会去实例化这个变量。
   - RDD中lazy延迟加载，需要action算子才会执行操作。

5. 执行流程

   - Spark会将对应的transformations组成一个pipeline，然后将这些pipeline分解成一系列的task去执行。

6. RDD模型的优势

   | 概念           | RDDs                                   | Distribute shared memory(分布式共享内存)              |
   | -------------- | -------------------------------------- | ----------------------------------------------------- |
   | Reads          | 粗粒度或者细粒度                       | 细粒度                                                |
   | Writes         | 粗粒度                                 | 细粒度                                                |
   | 数据一致性     | 不重要的（因为RDD是不可变的）          | 取决于app 或者 runtime                                |
   | 容错           | 利用lineage达到细粒度且低延迟的容错    | 需要应用checkpoints（就是需要写磁盘）并且需要程序回滚 |
   | 计算慢的任务   | 可以利用备份的任务来解决               | 很难做到                                              |
   | 计算数据的位置 | 基于数据本地性自动实现                 | 基于数据本地性自动实现                                |
   | 内存不足的行为 | 和已经存在的数据流处理系统一样，写磁盘 | 非常糟糕的性能（需要内存的交换？）                    |

7. RDDs不适用的应用

   - RDDs 不太适合用于需要异步且细粒度更新共享状态的应用，比如一个 web 应用或者数据递增的 web 爬虫应用的存储系统。

8. RDDs表达

   1. 三个列表、两个分区
      - partitions() :返回一个分区对象的列表。
      - preferredLocations(p) : 分区p数据存储在哪些机器节点中。
      - dependencies() :返回一个依赖列表。
      - iterator(p,parentlters) ：根据父亲分区的数据输入计算分区p的所有数据。
      - partitioner() : 返回这个RDD是hash还是range分区的元数据信息。
   2. Spark中表达RDDs的接口
      - 窄依赖，表示父亲RDDs的一个分区最多被子RDDs一个分区所依赖。
      - 宽依赖，表示父亲RDDs的一个分区可以被子RDDs多个分区所依赖。
      - 窄依赖可以使得在集群中一个机器节点上面流水线式执行所有父亲的分区数据。宽依赖需要父亲 RDDs 的所有分区数据准备好并且利用类似于 MapReduce 的操作将数据在不同的节点之间进行Shuffle。
      - 窄依赖从一个失败节点中恢复是非常高效的，因为只需要重新计算**相对应的父亲的分区数据**就可以，而且这个重新计算是在不同的节点进行并行重计算的，与此相反，在一个含有宽依赖的血缘关系 RDDs 图中，一个节点的失败可能导致一些分区数据的丢失，但是我们需要**重新计算父 RDD 的所有分区的数据**。
   3. 实现
      - HDFS files：抽样的输入 RDDs 是 HDFS 中的文件。对于这些 RDDs，partitions 返回文件中每一个数据块对应的一个分区信息（数据块的位置信息存储在 Partition 对象中），preferredLocations 返回每一个数据块所在的机器节点信息，最后 iterator 负责数据块的读取操作。
   4. 特性
      - 区分宽依赖和窄依赖其实就是遵循最大化pipeline的思想。

## 二、DAG流水线

1、问题

1. 我们今天说了，DAG 以 Shuffle 为边界划分 Stages，那你知道 Spark 是根据什么来判断一个操作是否会引入 Shuffle 的呢？

   ```
   rdd 会有 dep 属性，用来区分是否是 shuffle 生成的 rdd. 而 dep 属性的确定主要是根据子 rdd 是否依赖父 rdd 的某一部分数据，这个就得看他两的分区器(如果 tranf/action 有的话)。如果分区器一致，就不会产生 shuffle。
   ```

2. 在 Spark 中，同一 Stage 内的所有算子会融合为一个函数。你知道这一步是怎么做到的吗？

   ```
   在 task 启动后，会调用 rdd iterator 进行算子链的递归生成，调用 stage 图中最后一个 rdd 的 compute 方法，一般如果是 spark 提供的 rdd，compute 函数大都会继续调用父 rdd 的 iterator 方法，直到到 stage 的根 rdd，一般都是 sourceRdd，比如 hadoopRdd，KakaRdd，就会返回 source iterator。开始返回，如果子rdd 是 map 转换的，就会组成 itr.map(f)。如果再下一个是 filter 转换，就会组成 itr.map(f1).filter(f2)，以此类推。
   ```

## 三、调度系统

1、职能

- 将 DAG 拆分为不同的运行阶段 Stages；	--- DAGScheduler
- 创建分布式任务 Tasks 和任务组 TaskSet；  --- DAGScheduler
- 获取集群内可用的硬件资源情况；  --- SchedulerBackend
- 按照调度规则决定优先调度哪些任务 / 组； --- TaskScheduler
- 依序将分布式任务分发到执行器 Executor。 --- SchedulerBackend

2、DAGScheduler

3、SchedulerBackend

4、TaskScheduler

四、Spark面试题

```
面试题：spark去掉内存和mr比，谁快
答：

1.首先需要声明MapReduce和Spark都是基于内存进行运算的，真正的区别在于多个作业之间的数据通信，MapReduce的数据通信是基于磁盘IO的，而spark是基于内存的；

2.无论业务处理中是否需要排序，MapReduce的shuffle都是必须要经过排序的，而Spark的HashShuffle或者SortShuffle的bypass机制可以在不必要的排序场景下避免排序。

3.当数据处理流程中存在多个map和多个Reduce操作混合执行时，MapReduce只能提交多个Job执行，而Spark可以只提交一次，在一个任务中完成。

4.MapReduce 每次shuffle 操作后，必须写到磁盘，而 Spark 在 shuffle 后不一定落盘，如果Shuffle后的数据是需要反复用到的，则可以cache到内存中，方便迭代时使用。

5.spark task是基于fork线程的，启动时间较短；而MapReduce是基于进程的，启动时间较长；


Spark 的 DAG（有向无环图），这个 DAG 就相当于改进版的 MapReduce，它可以说是由多个 MapReduce 组成，当数据处理流程中存在多个map和多个Reduce操作混合执行时，MapReduce只能提交多个Job执行，而Spark可以只提交一次，在一个任务中完成。
这就导致了 MapReduce 会存在多次耗时的资源申请和资源释放，另外 MapReduce 每次shuffle 操作后，必须写到磁盘，而 Spark 在 shuffle 后不一定落盘，如果Shuffle后的数据是需要反复用到的，则可以cache到内存中，方便迭代时使用，所以Spark对于需要对数据进行反复迭代的操作（比如跑机器学习算法或者有中间结果的复杂计算等）是非常友好的。
```

## 四、存储系统

1、存储系统的主要三个应用方面

- RDD缓存
  - 截断DAG，可以降低失败重试的计算开销
  - 通过缓存内容访问，可以有效减少从头计算的次数，从整体上提升作业端到端的执行性能。
- Shuffle
  - Shuffle Write按照Reducer分区规则将中间数据写入本地磁盘。
  - Shuffle Reader从各个节点下载数据分片，并根据需要进行聚合计算。
- 广播变量

2、存储系统的组件

- BlockManager
  - Executors 端负责统一管理和协调数据的本地存取与跨节点传输,实现数据的存取、收发。
- BlockManagerMaster
  - BlockManager 与 Driver 端的 BlockManagerMaster 通信，不仅定期向 BlockManagerMaster 汇报本地数据元信息，还会不定时按需拉取全局数据存储状态。另外，不同 Executors 的 BlockManager 之间也会以 Server/Client 模式跨节点推送和拉取数据块。
- MemoryStore
- DiskStore
- DiskBlockManager

3、MemoryStore和DiskStore决定了数据存储在哪里的问题

1. Spark存储系统支持的两种类型：对象值（Object Values）、字节数组（Byte Array）
2. RDD缓存中的MemoryStore
   1. 统一采用MemoryEntry数据抽象对他们进行封装，具有两个实现类：DeserializedMemoryEntry和SerianlizedMemoryEntry，分别封装原始对象值和序列化的字节数组。
   2. DeserializedMemoryEntry:Array[T] 、SerianlizedMemoryEntry:ByteBuffer
   3. MemoryStore进行存储与访问的高效数据结构
      1. LinkedHashMap[BlockId, MemoryEntry]
      2. RDD缓存的三个步骤
         1. 第一步就是通过调用 putIteratorAsValues 或是 putIteratorAsBytes 方法，把 RDD 迭代器展开为数据值，然后把这些数据值暂存到一个叫做 ValuesHolder 的数据结构里。
         2. 为了节省内存开销，我们可以在存储数据值的 ValuesHolder 上直接调用 toArray 或是 toByteBuffer 操作，把 ValuesHolder 转换为 MemoryEntry 数据结构。
         3. 这些包含 RDD 数据值的 MemoryEntry 和与之对应的 BlockId，会被一起存入 Key 为 BlockId、Value 是 MemoryEntry 引用的链式哈希字典中。
      3. 内存不够:LRU算法
3. Shuffle中的DiskStore
   1. DiskStore 中数据的存取本质上就是字节序列与磁盘文件之间的转换。
   2. DiskBlockManager 的主要职责就是，记录逻辑数据块 Block 与磁盘文件系统中物理文件的对应关系，每个 Block 都对应一个磁盘文件。
   3. Spark 默认采用 SortShuffleManager 来管理 Stages 间的数据分发，在 Shuffle write 过程中，有 3 类结果文件：temp_shuffle_XXX、shuffle_XXX.data 和 shuffle_XXX.index。

