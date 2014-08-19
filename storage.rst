Storage相关配置参数
================

spark.local.dir
-------------------

这个看起来很简单，就是Spark用于写中间数据，如RDD Cache，Shuffle，Spill等数据的位置，那么有什么可以注意的呢。

首先，最基本的当然是我们可以配置多个路径（用逗号分隔）到多个磁盘上增加整体IO带宽，这个大家都知道。

其次，目前的实现中，Spark是通过对文件名采用hash算法分布到多个路径下的目录中去，如果你的存储设备有快有慢，比如SSD+HDD混合使用，那么你可以通过在SSD上配置更多的目录路径来增大它被Spark使用的比例，从而更好地利用SSD的IO带宽能力。当然这只是一种变通的方法，终极解决方案还是应该像目前HDFS的实现方向一样，让Spark能够感知具体的存储设备类型，针对性的使用。

需要注意的是，在Spark 1.0 以后，SPARK_LOCAL_DIRS (Standalone, Mesos) or LOCAL_DIRS (YARN)参数会覆盖这个配置。比如Spark On YARN的时候，Spark Executor的本地路径依赖于Yarn的配置，而不取决于这个参数。


spark.executor.memory
--------------------------------

Executor 内存的大小，和性能本身当然并没有直接的关系，但是几乎所有运行时性能相关的内容都或多或少间接和内存大小相关。这个参数最终会被设置到Executor的JVM的heap尺寸上，对应的就是Xmx和Xms的值

理论上Executor 内存当然是多多益善，但是实际受机器配置，以及运行环境，资源共享，JVM GC效率等因素的影响，还是有可能需要为它设置一个合理的大小。 多大算合理，要看实际情况

Executor的内存基本上是Executor内部所有任务共享的，而每个Executor上可以支持的任务的数量取决于Executor所管理的CPU Core资源的多少，因此你需要了解每个任务的数据规模的大小，从而推算出每个Executor大致需要多少内存即可满足基本的需求。

如何知道每个任务所需内存的大小呢，这个很难统一的衡量，因为除了数据集本身的开销，还包括算法所需各种临时内存空间的使用，而根据具体的代码算法等不同，临时内存空间的开销也不同。但是数据集本身的大小，对最终所需内存的大小还是有一定的参考意义的。

通常来说每个分区的数据集在内存中的大小，可能是其在磁盘上源数据大小的若干倍（不考虑源数据压缩，Java对象相对于原始裸数据也还要算上用于管理数据的数据结构的额外开销），需要准确的知道大小的话，可以将RDD cache在内存中，从BlockManager的Log输出可以看到每个Cache分区的大小（其实也是估算出来的，并不完全准确）

如： BlockManagerInfo: Added rdd_0_1 on disk on sr438:41134 (size: 495.3 MB)


反过来说，如果你的Executor的数量和内存大小受机器物理配置影响相对固定，那么你就需要合理规划每个分区任务的数据规模，例如采用更多的分区，用增加任务数量（进而需要更多的批次来运算所有的任务）的方式来减小每个任务所需处理的数据大小。


spark.storage.memoryFraction
------------------------------------------

如前面所说spark.executor.memory决定了每个Executor可用内存的大小，而spark.storage.memoryFraction则决定了在这部分内存中有多少可以用于Memory Store管理RDD Cache数据，剩下的内存用来保证任务运行时各种其它内存空间的需要。

spark.executor.memory默认值为0.6，官方文档建议这个比值不要超过JVM Old Gen区域的比值。这也很容易理解，因为RDD Cache数据通常都是长期驻留内存的，理论上也就是说最终会被转移到Old Gen区域（如果该RDD还没有被删除的话），如果这部分数据允许的尺寸太大，势必把Old Gen区域占满，造成频繁的FULL GC。

如何调整这个比值，取决于你的应用对数据的使用模式和数据的规模，粗略的来说，如果频繁发生Full GC，可以考虑降低这个比值，这样RDD Cache可用的内存空间减少（剩下的部分Cache数据就需要通过Disk Store写到磁盘上了），会带来一定的性能损失，但是腾出更多的内存空间用于执行任务，减少Full GC发生的次数，反而可能改善程序运行的整体性能


spark.streaming.blockInterval
----------------------------------------

这个参数用来设置Spark Streaming里Stream Receiver生成Block的时间间隔，默认为200ms。具体的行为表现是具体的Receiver所接收的数据，每隔这里设定的时间间隔，就从Buffer中生成一个StreamBlock放进队列，等待进一步被存储到BlockManager中供后续计算过程使用。理论上来说，为了每个Streaming Batch 间隔里的数据是均匀的，这个时间间隔当然应该能被Batch的间隔时间长度所整除。总体来说，如果内存大小够用，Streaming的数据来得及处理，这个blockInterval时间间隔的影响不大，当然，如果数据Cache Level是Memory+Ser，即做了序列化处理，那么BlockInterval的大小会影响序列化后数据块的大小，对于Java 的GC的行为会有一些影响。

此外spark.streaming.blockQueueSize决定了在StreamBlock被存储到BlockMananger之前，队列中最多可以容纳多少个StreamBlock。默认为10，因为这个队列Poll的时间间隔是100ms，所以如果CPU不是特别繁忙的话，基本上应该没有问题。


