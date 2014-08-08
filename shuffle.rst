Shuffle 相关配置参数
====================

Shuffle操作大概是对Spark性能影响最大的步骤之一（因为可能涉及到排序，磁盘IO，网络IO等众多CPU或IO密集的操作），这也是为什么在Spark 1.1的代码中对整个Shuffle框架代码进行了重构，将Shuffle相关读写操作抽象封装到Pluggable的Shuffle Manager中，便于试验和实现不同的Shuffle功能模块。例如为了解决Hash Based的Shuffle Manager在文件读写效率方面的问题而实现的Sort Base的Shuffle Manager。

spark.shuffle.manager
------------------------------ 

用来配置所使用的Shuffle Manager，目前可选的Shuffle Manager包括老的默认的org.apache.spark.shuffle.sort.HashShuffleManager和新的org.apache.spark.shuffle.sort.SortShuffleManager。

这两个ShuffleManager如何选择呢，首先需要了解他们在实现方式上的区别。

HashShuffleManager，故名思义也就是在Shuffle的过程中写数据时不做排序操作，只是将数据根据Hash的结果，将各个Reduce分区的数据写到各自的磁盘文件中。带来的问题就是如果Reduce分区的数量比较大的话，将会产生大量的磁盘文件。如果文件数量特别巨大，对文件读写的性能会带来比较大的影响，此外由于同时打开的文件句柄数量众多，序列化，以及压缩等操作需要分配的临时内存空间也可能会迅速膨胀到无法接受的地步，对内存的使用和GC带来很大的压力。

SortShuffleManager，在写入分区数据的时候，首先会根据实际情况对数据采用不同的方式进行排序操作，底线是至少按照Reduce分区Partition进行排序，这样来至于同一个Map任务Shuffle到不同的Reduce分区中去的所有数据都可以写入到同一个外部磁盘文件中去，用简单的Offset标志不同Reduce分区的数据在这个文件中的偏移量。这样一个Map任务就只需要生成一个shuffle文件，从而避免了上述HashShuffleManager可能遇到的问题

两者的性能比较，取决于内存，排序，文件操作等因素的综合影响。

对于不需要进行排序的Shuffle操作来说，如repartition等，如果文件数量不是特别巨大，HashShuffleManager面临的内存问题不大，而SortShuffleManager需要额外的根据Partition进行排序，显然HashShuffleManager的效率会更高。

而对于本来就需要在Map端进行排序的Shuffle操作来说，如ReduceByKey等，使用HashShuffleManager虽然在写数据时不排序，但在其它的步骤中仍然需要排序，而SortShuffleManager则可以将写数据和排序两个工作合并在一起执行，因此即使不考虑HashShuffleManager的内存使用问题，SortShuffleManager依旧可能更快。


spark.shuffle.consolidateFiles
----------------------------------------


这个配置参数仅适用于HashShuffleMananger的实现，同样是为了解决生成过多文件的问题，采用的方式是在不同批次运行的Map任务之间重用Shuffle输出文件，也就是说合并的是不同批次的Map任务的输出数据，但是每个Map任务所需要的文件还是取决于Reduce分区的数量，因此，它并不减少同时打开的输出文件的数量，因此对内存使用量的减少并没有帮助。只是HashShuffleManager里的一个折中的解决方案。

需要注意的是，这部分的代码实现尽管原理上说很简单，但是涉及到底层具体的文件系统的实现和限制等因素，例如在并发访问等方面，需要处理的细节很多，因此一直存在着这样那样的bug或者问题，导致在例如EXT3上使用时，特定情况下性能反而可能下降，因此从Spark 0.8的代码开始，一直到Spark 1.1的代码为止也还没有被标志为Stable，不是默认采用的方式。此外因为并不减少同时打开的输出文件的数量，因此对性能具体能带来多大的改善也取决于具体的文件数量的情况。所以是否使用，在什么版本中可以使用，最好还是根据你自己的实际情况测试以后决定。


spark.shuffle.spill / spark.shuffle.memoryFraction
--------------------------------------------------------------------

shuffle的过程中，如果涉及到排序，聚合等操作，势必会需要在内存中维护一些数据结构，进而占用额外的内存。如果内存不够用怎么办，那只有两条路可以走，一就是out of memory 出错了，二就是将部分数据临时写到外部存储设备中去，最后再合并到最终的Shuffle输出文件中去。

这里spark.shuffle.spill 决定是否Spill到外部存储设备（默认打开），而spark.shuffle.memoryFraction决定了当Shuffle过程中使用的内存达到总内存多少比例的时候开始Spill。

如果你的内存足够使用，当然也就不需要Spill，毕竟Spill带来了额外的磁盘操作，如果需要Spill，则可以通过spark.shuffle.memoryFraction调整Spill之前Shuffle占用内存的大小，进而调整Spill的频率和GC的行为。总的来说，如果Spill太过频繁，可以适当增加spark.shuffle.memoryFraction的大小，增加用于Shuffle的内存，减少Spill的次数。当然这样一来为了避免内存溢出，对应的可能需要减少RDD cache占用的内存，即减小spark.storage.memoryFraction的值，这样RDD cache的容量减少，有可能带来性能影响，因此需要综合考虑。

