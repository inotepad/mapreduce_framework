#ReduceTask的流程
***

ReduceTask reducer reduce之间的关系

* ReduceTask类似于OS中一个进程
* reducer仅仅是其中一个专门处理数据的对象,那么还需要输入数据和输出数据的对象
* reduce就是reducer如何处理数据的一个方法
* 但是ReduceTask不像MapTask那样简单，ReduceTask包含有三个阶段
 * COPY Phase
 * SORT Phase
 * REDUCE Phase

***
###ReduceTask中的流程
***

```java
  在类初始化的时候，
  setPhase(TaskStatus.Phase.SHUFFLE);
  
  之后运行的run函数
  public void run(...){
    //copyPhase
    copyPhase.complete();// copy is already complete
    //sortPhase
    sortPhase.complete();// sort is complete
    runNewReducer(...);
  }
  void runNewReducer(... ) {
    //初始化输入、reducer、输出对象
    reducer.run(reducerContext);
  }

```
* MapTask中就只有MAP Phase
* ReduceTask集中了三个阶段
 * COPY Phase
 * SORT Phase
 * REDUCE Phase

***
###COPY Phase
***
```java
    ReduceTask中的run方法部分代码
    boolean isLocal = "local".equals(job.get("mapred.job.tracker", "local"));
    if (!isLocal) {
      reduceCopier = new ReduceCopier(umbilical, job, reporter);
      if (!reduceCopier.fetchOutputs()) {
        if(reduceCopier.mergeThrowable instanceof FSError) {
          throw (FSError)reduceCopier.mergeThrowable;
        }
        throw new IOException("Task: " + getTaskID() + 
            " - The reduce copier failed", reduceCopier.mergeThrowable);
      }
    }
    copyPhase.complete();                         // copy is already complete
```
* 如果不是local执行的模式，那么需要使用ReduceCopier从其他机器上将map处理完的数据复制过来
* fetchOutputs方法是复制数据的主要实现
* 和其他机器通信使用的是HTTP协议
* 复制过来的数据，部分存在硬盘中，部分存在内存中

***
###SORT Phase
***
```
ReduceTask中的run方法
    final FileSystem rfs = FileSystem.getLocal(job).getRaw();
    RawKeyValueIterator rIter = isLocal
      ? Merger.merge(job, rfs, job.getMapOutputKeyClass(),
          job.getMapOutputValueClass(), codec, getMapFiles(rfs, true),
          !conf.getKeepFailedTaskFiles(), job.getInt("io.sort.factor", 100),
          new Path(getTaskID().toString()), job.getOutputKeyComparator(),
          reporter, spilledRecordsCounter, null)
      : reduceCopier.createKVIterator(job, rfs, reporter);
        
    // free up the data structures
    mapOutputFilesOnDisk.clear();
    sortPhase.complete();
```
* 将数据都复制过来之后，每个map处理之后的数据都是一个文件
* 需要将这些数据再次封装成KV

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
* 之后如何获取这些键值对并且处理呢？  注意： reduce()处理(K,List&lt;V>)形式的**一个**键值对
* 但是每个map处理之后的数据中肯定非常多而且都是(K,V)形式的，那么是如何把处理所有的键值呢？
* 这里回答如何不断获取，下面一节回答如何(K,V)->(K,List&lt;V>)
* Reducer在运行的时候不断的获得键值对，不断的交给reduce去处理。

###总结
* COPY阶段将所有分散在其他机器上的map处理过的数据收集过来
* SORT将这么多的文件组织成KV对
* reducer负责处理数据
