.. contents::

通用设置
--------

spark.serializer
~~~~~~~~~~~~~~~~

默认为org.apache.spark.serializer.JavaSerializer, 可选 org.apache.spark.serializer.KryoSerializer, 实际上只要是org.apache.spark.serializer的子类就可以了,不过如果只是应用,大概你不会自己去实现一个的。

序列化对于spark应用的性能来说,还是有很大影响的,号称在特定的数据格式的情况下,KryoSerializer的性能可以达到JavaSerializer的10倍以上,当然放到整个Spark程序中来考量,比重就没有那么大了,但是以Wordcount为例，通常也很容易达到30%以上的性能提升。而对于一些Int之类的基本类型数据，性能的提升就几乎可以忽略了。KryoSerializer依赖Twitter的Chill库来实现，相对于JavaSerializer，主要的问题在于不是所有的Java Serializable对象都能支持。

需要注意的是，这里可配的Serializer针对的对象是Shuffle数据，以及RDD Cache等场合，而Spark Task的序列化是通过spark.closure.serializer来配置，但是目前只支持JavaSerializer，所以等于没法配置啦。

更多Kryo序列化相关优化配置,可以参考 http://spark.apache.org/docs/latest/tuning.html#data-serialization 一节


spark.local.dir
~~~~~~~~~~~~~~~

这个看起来很简单，就是Spark用于写中间数据，如RDD Cache，Shuffle，Spill等等数据的位置。那么有什么可以注意的呢。

首先，最基本的当然是我们可以配置多个路径（用逗号分隔）到多个磁盘上增加整体IO带宽了。

其次，目前的实现中，Spark是通过对文件名采用hash算法分布到多个路径下的目录中去，如果你的存储设备有快有慢，比如SSD+HDD混合使用，那么你可以通过在SSD上配置更多的目录路径来增大它被Spark使用的比例，从而更好地利用SSD的IO带宽能力。当然这只是一种变通的方法，终极解决方案还是应该像目前HDFS的实现方向一样，让Spark能够Aware具体的存储设备类型。

最后在Spark 1.0 以后，SPARK_LOCAL_DIRS (Standalone, Mesos) or LOCAL_DIRS (YARN)参数会覆盖这个配置，基本上Spark On YARN的时候，Spark Executor的本地路径依赖于Yarn的配置，而不取决于这个参数。


spark.executor.memory
~~~~~~~~~~~~~~~~~~~~~

Executor 内存的大小，和性能本身当然并没有直接的关系，但是几乎所有运行时性能相关的内容还是或多或少间接和它相关的。这个参数最终会被设置到Executor的JVM的heap尺寸上，对应的就是Xmx和Xms的值

理论上Executor 内存当然是多多益善，但是实际受机器配置，以及运行环境，资源共享，JVM GC效率等因数的影响，还是有可能需要为它设置一个合理的大小。 多大算合理，要看实际情况，我只能说你需要了解和考量的因素包括：

Executor的内存基本上是Task共享的，而每个Executor上可以支持的Task的数量取决于Executor所管理的CPU Core资源的多少，因此你需要了解每个Task的数据规模的大小，从而推算出每个Executor大致需要多少内存即可满足基本的需求。

如何知道每个任务所需内存的大小呢，这个很难统一的衡量，因为根据具体的代码算法等不同，除了数据集本身的开销，还包括算法所需各种临时内存空间的使用。但是数据集本身的大小，对最终所需内存的大小还是有一定的参考意义的。

通常来说每个分区的数据集在内存中的大小，可能是其在磁盘上源数据大小的若干倍（不考虑源数据压缩，Java对象相对于原始裸数据也还要算上用于管理数据的数据结构的额外开销），需要准确的知道大小的话，可以根据将RDD cache在内存中，从BlockManager的Log输出可以看到每个Cache分区的大小（其实也是估算出来的，并不完全准确）

反过来说，如果你的Executor的数量和内存大小受机器物理配置影响相对固定，那么你就需要合理规划每个分区任务的数据规模，例如采用更多的分区，用增加任务数量（进而需要更多的批次来运算所有的任务）的方式来减小每个任务所需处理的数据大小。


spark.storage.memoryFraction
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

如前面所说spark.executor.memory决定了每个Executor可用内存的大小，而spark.storage.memoryFraction则决定了在这部分内存中有多少可以用于Memory Store管理RDD Cache数据，剩下的内存用来保证任务运行时各种其它内存空间的需要。

spark.executor.memory默认值为0.6，官方文档建议这个比值不要超过JVM Old Gen区域的比值。这也很容易理解，因为RDD Cache数据通常都是长期驻留内存的，理论上也就是说最终会被转移到Old Gen区域（如果该RDD还没有被删除的话），如果这部分数据允许的尺寸太大，势必把Old Gen区域占满，造成频繁的FULL GC。

如何调整这个比值，取决于你的应用对数据的使用模式和数据的规模，粗略的来说，如果频繁发生Full GC，可以考虑降低这个比值，这样RDD Cache可用的内存空间减少（剩下的部分Cache数据就需要通过Disk Store写到磁盘上了），会带来一定的性能损失，但是腾出更多的内存空间用于执行任务，减少Full GC发生的次数，反而可能改善程序运行的整体性能



