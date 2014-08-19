Shuffle 相关
===========

Shuffle操作大概是对Spark性能影响最大的步骤之一（因为可能涉及到排序，磁盘IO，网络IO等众多CPU或IO密集的操作），这也是为什么在Spark 1.1的代码中对整个Shuffle框架代码进行了重构，将Shuffle相关读写操作抽象封装到Pluggable的Shuffle Manager中，便于试验和实现不同的Shuffle功能模块。例如为了解决Hash Based的Shuffle Manager在文件读写效率方面的问题而实现的Sort Base的Shuffle Manager。

spark.shuffle.manager
------------------------------ 

用来配置所使用的Shuffle Manager，目前可选的Shuffle Manager包括默认的org.apache.spark.shuffle.sort.HashShuffleManager（配置参数值为hash）和新的org.apache.spark.shuffle.sort.SortShuffleManager（配置参数值为sort）。

这两个ShuffleManager如何选择呢，首先需要了解他们在实现方式上的区别。

HashShuffleManager，故名思义也就是在Shuffle的过程中写数据时不做排序操作，只是将数据根据Hash的结果，将各个Reduce分区的数据写到各自的磁盘文件中。带来的问题就是如果Reduce分区的数量比较大的话，将会产生大量的磁盘文件。如果文件数量特别巨大，对文件读写的性能会带来比较大的影响，此外由于同时打开的文件句柄数量众多，序列化，以及压缩等操作需要分配的临时内存空间也可能会迅速膨胀到无法接受的地步，对内存的使用和GC带来很大的压力，在Executor内存比较小的情况下尤为突出，例如Spark on Yarn模式。

SortShuffleManager，是1.1版本之后实现的一个试验性（也就是一些功能和接口还在开发演变中）的ShuffleManager，它在写入分区数据的时候，首先会根据实际情况对数据采用不同的方式进行排序操作，底线是至少按照Reduce分区Partition进行排序，这样来至于同一个Map任务Shuffle到不同的Reduce分区中去的所有数据都可以写入到同一个外部磁盘文件中去，用简单的Offset标志不同Reduce分区的数据在这个文件中的偏移量。这样一个Map任务就只需要生成一个shuffle文件，从而避免了上述HashShuffleManager可能遇到的文件数量巨大的问题

两者的性能比较，取决于内存，排序，文件操作等因素的综合影响。

对于不需要进行排序的Shuffle操作来说，如repartition等，如果文件数量不是特别巨大，HashShuffleManager面临的内存问题不大，而SortShuffleManager需要额外的根据Partition进行排序，显然HashShuffleManager的效率会更高。

而对于本来就需要在Map端进行排序的Shuffle操作来说，如ReduceByKey等，使用HashShuffleManager虽然在写数据时不排序，但在其它的步骤中仍然需要排序，而SortShuffleManager则可以将写数据和排序两个工作合并在一起执行，因此即使不考虑HashShuffleManager的内存使用问题，SortShuffleManager依旧可能更快。

spark.shuffle.sort.bypassMergeThreshold
--------------------------------------------------

这个参数仅适用于SortShuffleManager，如前所述，SortShuffleManager在处理不需要排序的Shuffle操作时，由于排序带来性能的下降。这个参数决定了在这种情况下，当Reduce分区的数量小于多少的时候，在SortShuffleManager内部不使用Merge Sort的方式处理数据，而是与Hash Shuffle类似，直接将分区文件写入单独的文件，不同的是，在最后一步还是会将这些文件合并成一个单独的文件。这样通过去除Sort步骤来加快处理速度，代价是需要并发打开多个文件，所以内存消耗量增加，本质上是相对HashShuffleMananger一个折衷方案。 这个参数的默认值是200个分区，如果内存GC问题严重，可以降低这个值。


spark.shuffle.consolidateFiles
----------------------------------------

这个配置参数仅适用于HashShuffleMananger的实现，同样是为了解决生成过多文件的问题，采用的方式是在不同批次运行的Map任务之间重用Shuffle输出文件，也就是说合并的是不同批次的Map任务的输出数据，但是每个Map任务所需要的文件还是取决于Reduce分区的数量，因此，它并不减少同时打开的输出文件的数量，因此对内存使用量的减少并没有帮助。只是HashShuffleManager里的一个折中的解决方案。

需要注意的是，这部分的代码实现尽管原理上说很简单，但是涉及到底层具体的文件系统的实现和限制等因素，例如在并发访问等方面，需要处理的细节很多，因此一直存在着这样那样的bug或者问题，导致在例如EXT3上使用时，特定情况下性能反而可能下降，因此从Spark 0.8的代码开始，一直到Spark 1.1的代码为止也还没有被标志为Stable，不是默认采用的方式。此外因为并不减少同时打开的输出文件的数量，因此对性能具体能带来多大的改善也取决于具体的文件数量的情况。所以即使你面临着Shuffle文件数量巨大的问题，这个配置参数是否使用，在什么版本中可以使用，也最好还是实际测试以后再决定。

spark.shuffle.spill 
-----------------------

shuffle的过程中，如果涉及到排序，聚合等操作，势必会需要在内存中维护一些数据结构，进而占用额外的内存。如果内存不够用怎么办，那只有两条路可以走，一就是out of memory 出错了，二就是将部分数据临时写到外部存储设备中去，最后再合并到最终的Shuffle输出文件中去。

这里spark.shuffle.spill 决定是否Spill到外部存储设备（默认打开）,如果你的内存足够使用，或者数据集足够小，当然也就不需要Spill，毕竟Spill带来了额外的磁盘操作。

spark.shuffle.memoryFraction / spark.shuffle.safetyFraction
--------------------------------------------------------------------

在启用Spill的情况下，spark.shuffle.memoryFraction（1.1后默认为0.2）决定了当Shuffle过程中使用的内存达到总内存多少比例的时候开始Spill。

通过spark.shuffle.memoryFraction可以调整Spill的触发条件，即Shuffle占用内存的大小，进而调整Spill的频率和GC的行为。总的来说，如果Spill太过频繁，可以适当增加spark.shuffle.memoryFraction的大小，增加用于Shuffle的内存，减少Spill的次数。当然这样一来为了避免内存溢出，对应的可能需要减少RDD cache占用的内存，即减小spark.storage.memoryFraction的值，这样RDD cache的容量减少，有可能带来性能影响，因此需要综合考虑。

由于Shuffle数据的大小是估算出来的，一来为了降低开销，并不是每增加一个数据项都完整的估算一次，二来估算也会有误差，所以实际暂用的内存可能比估算值要大，这里spark.shuffle.safetyFraction（默认为0.8）用来作为一个保险系数，降低实际Shuffle使用的内存阀值，增加一定的缓冲，降低实际内存占用超过用户配置值的概率。

spark.shuffle.spill.compress / spark.shuffle.compress
--------------------------------

这两个配置参数都是用来设置Shuffle过程中是否使用压缩算法对Shuffle数据进行压缩，前者针对Spill的中间数据，后者针对最终的shuffle输出文件，默认都是True

理论上说，spark.shuffle.compress设置为True通常都是合理的，因为如果使用千兆以下的网卡，网络带宽往往最容易成为瓶颈。此外，目前的Spark任务调度实现中，以Shuffle划分Stage，下一个Stage的任务是要等待上一个Stage的任务全部完成以后才能开始执行，所以shuffle数据的传输和CPU计算任务之间通常不会重叠，这样Shuffle数据传输量的大小和所需的时间就直接影响到了整个任务的完成速度。但是压缩也是要消耗大量的CPU资源的，所以打开压缩选项会增加Map任务的执行时间，因此如果在CPU负载的影响远大于磁盘和网络带宽的影响的场合下，也可能将spark.shuffle.compress 设置为False才是最佳的方案

对于spark.shuffle.spill.compress而言，情况类似，但是spill数据不会被发送到网络中，仅仅是临时写入本地磁盘，而且在一个任务中同时需要执行压缩和解压缩两个步骤，所以对CPU负载的影响会更大一些，而磁盘带宽（如果标配12HDD的话）可能往往不会成为Spark应用的主要问题，所以这个参数相对而言，或许更有机会需要设置为False。


总之，Shuffle过程中数据是否应该压缩，取决于CPU/DISK/NETWORK的实际能力和负载，应该综合考虑。


