***
###Reducer运行的核心逻辑
***
```java
    public void run(Context context) throws IOException, InterruptedException {
        setup(context);
        while (context.nextKey()) {
          reduce(context.getCurrentKey(), context.getValues(), context);
       }
        cleanup(context);
    }
    默认的reduce方法
    protected void reduce(KEYIN key, Iterable<VALUEIN> values, Context context
                          ) throws IOException, InterruptedException {
        for(VALUEIN value: values) {
          context.write((KEYOUT) key, (VALUEOUT) value);
        }
    }
```
#####流程如下：
* setup方法做了一些配置，默认是空。
* 不断的读取下一个(K,List&lt;V>)，并交给reduce一个一个KV来处理
* 默认的reduce方法就是什么都不做，输入和输出时一样的
* 这里使用的getCurrentKey()和getCurrentValue()是对输入的封装。
* 这里使用的write方法是对输出对象的封装

#####以WordCount来举例
* 在COPY Phase和SORT Pahse时，map处理之后的数据已经收集好了
* 之后如何获取这些键值对并且处理呢？  注意： reduce()处理(K,List&lt;,V>)形式的**一个**键值对
* 但是每个map处理之后的数据中肯定非常多而且都是(K,V)形式的，那么是如何把处理所有的键值呢？
* 这里回答如何不断获取，下面一节回答如何(K,V)->(K,List&lt;V>)
* Reducer在运行的时候不断的获得键值对，不断的交给reduce去处理。

***
###输入的实例化
***
#####数据的来源
```
    ReduceTask中的run方法
    RawKeyValueIterator rIter = reduceCopier.createKVIterator(job, rfs, reporter);
    这个是Reducer的输入，就是将收集到的map输出封装成KV

    之后在runNewReducer中又对该Iterator进行了一些封装。
```
* 输入的数据源是获取的map处理之后的数据,分布在硬盘中的diskSegments和在内存中的InMemorySegments中
* 这些数据通过RawKVIteratorReader来读取
* createKVIterator得到的是MergeQueue，主要是对数据这些数据排序
* 注意比较mapper和reducer获取数据的不同
 * mapper中调用nextKeyValue判断是否还有下一个数据
 * reducer使用nextKey判断是否还有下一个数据
 * mapper中使用getCurrentKey和getCurrentValue获取数据
 * reducer中使用getCurrentKey和getValues获取数据
* 还有一个值得注意的地方是数据源中的数据是(K,V)形式的，而nextKey得到的则是(K,List&lt;V>)
* 那么如何发生的(K,V)--->(K,List&lt;,V>)呢？下面将解决这个问题

#####(K,V)--->(K,List&lt;V>)
Context继承了ReduceContext,实现的方法都在ReduceContext中

首先，如果自己实现大概会是这个样子的

* nextKey返回是否存在，并且如果有下一个KV，那么直接读取进来，保存起来
* 对于List&lt;V>的迭代步骤
 * 如果是第一个，直接返回之前读取的value
 * 在之后的读取中，要不断的判断这一个Key和之前的是否相同
 * 相同的话，直接返回就可以了
 * 如果不同呢？那岂不是多读出了这么一个KV对!!!
* 就是说，只要是进行过了迭代操作，那么就会多读一个KV对，难道要每次都检查是不是进行过迭代吗？
 * 有些应用可能会除了第一次循环之外，其他的每次循环之前都是进行过List&lt;V>迭代操作的
 * 但是有些应用可能会跳过一些特定的K值，不对List&lt;V>迭代
 * 迭代过的话，就直接返回当前保存的；某个Key被跳过，没有迭代过，要先跳过这个Key

就是说，不管怎么样都是会多读一个，毕竟要对比是不是相同的K，那么这里是如何解决这个问题的呢？（这里的实现重写了nextKeyValue方法）

* 将数据的输入分成了两层
* 第一层是输入的是一个Iterator input，让它里面的存储下一个KV，这里相当于是一个缓冲层
* 第二层是当前封装成的Iterator，有以下特点
 * 存储当前使用的KV
 * hasMore标示input中还有没有数据(就是下一个数据还有没有)
 * nextKeyIsSame标示下一个K是否和当前K相同

实际的使用：

* 初始化的时候调用input.next()，让input中有下一个使用的数据
* nextKey中调用nextKeyValue()，将数据从input中取出，并将下一个数据填充到input中
 * 这样key就是存储的当前使用的，value也是当前使用的
 * 对List &lt;V>迭代的时候，如果是读取第一个value，那么直接返回
 * 如果不是第一个value，那么调用nextKeyValue获取下一个V的值，再返回
 * 如果获取到了最后一个value，那么nextKeyIsSame变成了false，迭代结束
* 迭代结束的之后，input中存有下一个可以使用的KV，在nextKey中会被读取到，新的一轮又开始了

应该说，将输入的这个Iterator作为缓冲的中间层，非常优雅的解决了只有一层总是会多读一个KV的问题

***
###输出的实例化
***
```
     org.apache.hadoop.mapreduce.RecordWriter<OUTKEY,OUTVALUE> trackedRW = 
       new NewTrackingRecordWriter<OUTKEY, OUTVALUE>(reduceOutputCounter,
         job, reporter, taskContext);
```
* 这个其实就是对FileOutputFormat的封装
* FileOutputFormat和FileInputFormat比较类似，这里不再赘述
数据来自硬盘和内存
