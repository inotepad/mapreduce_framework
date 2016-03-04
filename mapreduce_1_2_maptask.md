***
#MapTask的数据处理流程
***

MapTask mapper map之间的关系

* MapTask类似于OS中一个进程
* mapper仅仅是其中一个专门处理数据的对象,那么还需要输入数据和输出数据的对象
* map就是mapper如何处理数据的一个方法

***
###MapTask的流程
***

#####MapTask的run方法

```
（来自org.apache.hadoop.mapred.MapTask）
  public void run(...) {
    //...一些初始化工作
    runNewMapper(...);
    //...精简代码
  }
```

* MapTask就是做一些初始化工作，然后调用runNewMapper方法
* 和名字一样，runNewMapper就是启动mapper
* runNewMapper中的New是指使用了新的API

#####runNewMapper方法的几个作用

```java
（来自org.apache.hadoop.mapred.MapTask）
    runNewMapper(...){
        //初始化输入输出操作等对象
        mapper.run(...);//处理数据
        //清理工作
    }
```

* 初始化各种对象
 * 输入
 * 输出
 * mapper
* 运行mapper来处理数据
* 最后清理工作

***
###mapper运行的核心逻辑
***

```java
（来自org.apache.hadoop.mapreduce.Mapper）
    public void run(Context context) throws IOException, InterruptedException {
        setup(context);
        while (context.nextKeyValue()) {
            map(context.getCurrentKey(), context.getCurrentValue(), context);
    }
    cleanup(context);
}
    默认的map方法
    protected void map(KEYIN key, VALUEIN value, 
                         Context context) throws IOException, InterruptedException {
        context.write((KEYOUT) key, (VALUEOUT) value);
    }
```
#####流程如下：
* setup方法做了一些配置，默认是空。
* 不断的读取下一个(K,V)，并交给map逐个处理KV
* 默认的map方法就是什么都不做，输入和输出时一样的
* 这里使用的getCurrentKey()是对RecordReader的getCurrentKey()的封装。
* 这里使用的getCurrentValue()是对RecordReader的getCurrentValue()对应封装。
* write方法是对输出对象的write方法的封装

#####以WordCount来举例
* 类LineReader可以从流中读取一行，类LineRecordReader将这一行变成一个键值对。
* 之后如何获取这些键值对并且处理呢？  注意： map()只处理**一个**键值对
* 但是每个split中肯定不止一行，那么是如何不断的读取每一行，生成一个个的键值对来处理呢？
* 答案就是这里，mapper在运行的时候不断的获得键值对，不断的交给map去处理。

***
###输入的实例化
***
#####InputFormat和RecordReader的区别
* InputFormat和RecordReader的分别初始化，感觉有点奇怪
* 因为InputFormat关注于文件如何分割，所以有isSplitable、getSplits方法
* 而RecordReader关注于将一个文件中的内容转换成了键值对

#####以WordCount来举例
* FileInputFormat将一个文件分成几个split
* LineRecordReader将文本封装为(K,V),K是偏移字节数，V是每行的内容。

#####InputFormat
```java
（来自org.apache.hadoop.mapred.MapTask）
runNewMapper()部分代码
    // make the input format
    org.apache.hadoop.mapreduce.InputFormat<INKEY,INVALUE> inputFormat =
      (org.apache.hadoop.mapreduce.InputFormat<INKEY,INVALUE>)
        ReflectionUtils.newInstance(taskContext.getInputFormatClass(), job);
```

* 这个主要是如何从一个文件分为split，生成一个输入对象
* 可以看出来，调用了getInputFormatClass，这样可以获取到我们自己设置的输入类
* 默认输入方法是TextInputFormat

#####RecordReader

```java
runNewMapper()中初始化RecordReader
    org.apache.hadoop.mapreduce.RecordReader<INKEY,INVALUE> input =
      new NewTrackingRecordReader<INKEY,INVALUE>
          (split, inputFormat, reporter, job, taskContext);
```

* NewTrackingRecordReader&lt;INKEY,INVALUE>其实是对获得(K,V)的方法又进行了封装，增加了一些
记录。
* 这个输入对象才能够读取文本，封装了键值对，提供nextKeyValue(),getCurrentkey()，getCurrentValue()等方法。

***
###输出的实例化
***
* map处理后的输出是放在缓冲区中

####RecordWriter

```java
（来自org.apache.hadoop.mapred.MapTask）
runNewMapper()中初始化RecordWriter
       // get an output object
      if (job.getNumReduceTasks() == 0) {
         output =
           new NewDirectOutputCollector(taskContext, job, umbilical, reporter);
      } else {
        output = new NewOutputCollector(taskContext, job, umbilical, reporter);
      }
```

#####NewOutputCollector中的write()

* 可以从map的运行中看出，调用的输出对象context的write方法,其实就是out的write方法

```
  public void write(K key, V value) throws IOException, InterruptedException {
      collector.collect(key, value,
                        partitioner.getPartition(key, value, partitions));
    }
```

其中

```
MapOutputCollector<K,V> collector;
collector = new MapOutputBuffer<K,V>(umbilical, job, reporter);
```

* 这里可以知道了最终是使用的MapOutputBuffer中的collect()，直接写入了缓存中
* 收集的数据就是(key,value,partition)三个数据作为一个逻辑单元。至于为什么要加入partition，之后再解释。
* 现在整体的架构已经清楚了，那么我们之后重点关注(K,V)在经过map处理之后又发生了什么。

###总结
* MapTask负责初始化各种输入、处理、输出的对象
* InputFormat对象和RecordReader对象负责将数据封装好输入
* mapper对象中不断的读取数据，通过map方法去处理
* map处理之后的数据，缓冲在MapOutputBuffer对象中
