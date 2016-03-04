#SORT Phase
***

* 在经过COPY Phase之后，部分文件在内存中，部分文件在硬盘中
* 显然这在不同的介质中，而且不是有序的
* 那么如何处理这些不同介质并且排序呢？
 
* 使用MergeQueue，边排序边输出(详细的的介绍在Merge中)
* 边给Reducer输入数据，边执行排序
* 全部排序好在输出会降低运行效率
* 此时需要满足磁盘上的文件个数小于10个，如果文件个数太多，那么还需要不断的Merge
* 并且内存中的数据数据量小于最大可用内存的X%，X是由mapred.job.reduce.input.buffer.percent指定

* 由于有些文件是在硬盘中，需要封装成KV对，才能让reduce来读取
* RawKVIteratorReader将硬盘中的文件封装成KV形式
