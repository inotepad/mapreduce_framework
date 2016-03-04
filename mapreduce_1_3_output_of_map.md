#map的输出结果
***
###数据在map中的流动
***
![output-of-map](_image/2.output-of-map.png)

* Partition就是实现给数据贴标签功能，标记该数据应该由哪个reducer去处理。
* Serialize
* 把数据写入内存的缓冲区

**以上是mainThread中write方法所做的几件事情，获取(K,V)->map->写入缓冲区,在这之中又夹杂了一些小的操作比如Partition什么的**

**下面的就是spillTread做的事情了**

* 如果mainThread发现内存缓冲区到某个临界条件了，那么就唤醒spillThread
* spillThread把内存中的数据sortAndSpill到硬盘上,每次spill都会生成一个文件（溢写文件）。
* 如果有设置combiner,那么先执行combine，再写入到硬盘上。

**如果所有数据都已经完成处理的时候，spillThread完成了使命，就应该退出**

* mainThread调用sortAndSpill()后，mergePart()会将多个溢写文件变成一个文件。

***
###Partition
***
```java
public void write(K key, V value) throws IOException, InterruptedException {
    collector.collect(key, value,
                      partitioner.getPartition(key, value, partitions));
}
```
* 上次说到，存入的数据是(key,value,partition)三个数据，那么partition是用来干什么的呢？
* 前面说过Split是为了分西瓜给MapTask吃，map输出的数据应该分给不同的reduce去处理，那么怎么标示这个数据分给哪个reduce呢？
* Partition就是实现给数据贴标签功能，就是再次切西瓜把西瓜分好,具体的实现就是使用数字来标记。

####Partitioner的实例化

```java
来自org.apache.hadoop.mapred.NewOutputCollector部分代码
 partitioner = (org.apache.hadoop.mapreduce.Partitioner<K,V>)
           ReflectionUtils.newInstance(jobContext.getPartitionerClass(), job);

来自org.apache.hadoop.mapreduce,JobContext
   public Class<? extends Partitioner<?,?>> getPartitionerClass() 
      throws ClassNotFoundException {
     return (Class<? extends Partitioner<?,?>>) 
      conf.getClass(PARTITIONER_CLASS_ATTR, HashPartitioner.class);
   }
```
* 默认的方法是HashPartition

***
###Serialize
***

* 该部分和写入缓冲区紧密相关，之后分析

***
###整体看MapOutputBuffer存储结构
***
![overview](_image/3.0.MapOutputBuffer.png)

* 这是一个两级索引(kvoffsets和kvindices)的环形缓冲区(数据在kvbuffer中)。
* 可以把第一级索引看做是引用，剩下的作为对象的数组来看待。
* kvoffsets、kvindices、kvbuffer就是三个大数组，其中kvbuffer非常大。
* 环形:kvnetx=(kvindex+1)%kvoffsets.length
* 该部分的管理特别复杂，之后由专门的分析
* 写入缓冲区就是上篇文章最后讲到的collect方法。
* 从缓冲区读取出有两种方法
 * 一种方法是sortAndSpill方法中使用的直接从内部的数组读取（spillThread是写在MapOutputBuffer的一个内部类）
 * 还有一种方法是有combiner使用，MRResultIterator将缓冲区内的数据组织成KV对的读取。
 
***
###Spill
***
* 如果内存缓冲区的数据到达一定的条件，就要把数据写入硬盘，以免占用太多内存
* spillThread调用的是sortAndSpill()
* sort就是先使用的是QuickSort()对内存缓冲区中需要写硬盘上的这部分数据进行排序
* 然后使用Writer写入硬盘。

####Sort时从内存缓冲区中读取数据(可以先跳过)

```java

  final int kvoff = kvoffsets[spindex % kvoffsets.length];
  getVBytesForOffset(kvoff, value);
  key.reset(kvbuffer, kvindices[kvoff + KEYSTART],
             (kvindices[kvoff + VALSTART] - 
                kvindices[kvoff + KEYSTART]));
```
####使用快排对数据进行排序

```java
排序使用的是QuickSort
      sorter = ReflectionUtils.newInstance(
                  job.getClass("map.sort.class", QuickSort.class, IndexedSorter.class), job);
比较函数
    public int compare(int i, int j) {
      final int ii = kvoffsets[i % kvoffsets.length];
      final int ij = kvoffsets[j % kvoffsets.length];
      // sort by partition
      if (kvindices[ii + PARTITION] != kvindices[ij + PARTITION]) {
        return kvindices[ii + PARTITION] - kvindices[ij + PARTITION];
      }
      // sort by key
      return comparator.compare(kvbuffer,
          kvindices[ii + KEYSTART],
          kvindices[ii + VALSTART] - kvindices[ii + KEYSTART],
          kvbuffer,
          kvindices[ij + KEYSTART],
          kvindices[ij + VALSTART] - kvindices[ij + KEYSTART]);
    }
交换函数
    public void swap(int i, int j) {
      i %= kvoffsets.length;
      j %= kvoffsets.length;
      int tmp = kvoffsets[i];
      kvoffsets[i] = kvoffsets[j];
      kvoffsets[j] = tmp;
    }
```

* 就一个排序过程来讲，其实就是需要两个操作，比较大小和交换位置。
* 这里排序的设计是相当精彩的。
* 从compare中看出来，先根据partition的大小来排序，再根据key来排序。
* 排序实际上仅仅是改变了kvoffsets中的数据，其他的没有任何的改变。其实也不能改变第二级索引，之后说明为什么。

####Writer写入硬盘

* 使用IFile类将数据写入硬盘

#####溢写文件的结构
![spillfile](_image/3.5.spill.png)

* 由于已经排好序，所以相同的Partition的数据是连续的，相同的K的数据也是连续的。
* Spill的过程，相同Partition的数据连续写入，相同Partition按照Key排序，不同Partition之间使用Partition排序。
* 数据文件中，相同Partition的文件是连续存放的，指定每一部分的长度就可以了。
* 就是说每个Partition有一个IndexRecord记录就可以。
* 有专门的索引文件来存储所有的索引。


####combiner
```java
            if (combinerRunner == null) {
              // spill directly
              }
            } else {
                combinerRunner.combine(kvIter, combineCollector);
              }
            }
```

* MRResultIterator类将缓冲区的读取变成了KV对的读取
* combiner输入数据是内存，输出是硬盘，也就是spill中间执行。
* combiner就可以从缓冲区中把内容读取出来，进行处理，写入文件。

***
###Merge
***

![Merge合并硬盘中的文件](_image/5.1.Merge.png)

* Spill过程会执行多次，生成多个溢写文件
* merge过程就是将这么多的溢写的文件合并成一个文件,并且保持以下性质
 * 之前的多个文件是先按照partition来排序，partition相同的则按照key来排序的
 * 新的文件是也是先按照partition来排序，partition相同的则按照key来排序的一个文件
 * 也就是说merge包含由排序和合并文件两个功能。
* 由于Merger在Map和Reduce中均有使用，而且非常复杂，之后做单独分析。
