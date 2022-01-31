# Spark Streaming Guide 

[https://spark.apache.org/docs/latest/streaming-programming-guide.html](https://spark.apache.org/docs/latest/streaming-programming-guide.html)

## **Overview**

Spark Streaming is an extension of the core Spark API that enables scalable, high-throughput, fault-tolerant stream processing of live data streams. Data can be ingested from many sources like Kafka, Kinesis, or TCP sockets, and can be processed using complex algorithms expressed with high-level functions like `map`, `reduce`, `join` and `window`. Finally, processed data can be pushed out to filesystems, databases, and live dashboards. In fact, you can apply Spark’s [machine learning](https://spark.apache.org/docs/latest/ml-guide.html) and [graph processing](https://spark.apache.org/docs/latest/graphx-programming-guide.html) algorithms on data streams.

![](../images/spark_streaming_guide(1).png)

Internally, it works as follows. Spark Streaming receives live input data streams and divides the data into batches, which are then processed by the Spark engine to generate the final stream of results in batches.

![Untitled](../images/spark_streaming_guide(2).png)

Spark Streaming provides a high-level abstraction called *discretized stream* or *DStream*, which represents a continuous stream of data. DStreams can be created either from input data streams from sources such as Kafka, and Kinesis, or by applying high-level operations on other DStreams. Internally, a DStream is represented as a sequence of [RDDs](https://spark.apache.org/docs/latest/api/scala/org/apache/spark/rdd/RDD.html).

This guide shows you how to start writing Spark Streaming programs with DStreams. You can write Spark Streaming programs in Scala, Java or Python (introduced in Spark 1.2), all of which are presented in this guide. You will find tabs throughout this guide that let you choose between code snippets of different languages.

**Note:** There are a few APIs that are either different or not available in Python. Throughout this guide, you will find the tag **Python API** highlighting these differences.

- Spark Streaming 의 특징: scalable, high-throughput, fault-tolerant.
- Kafka, Kinesis, TCP Socket 등에서 들어오는 데이터들을 스파크에서 처리해주고 (map, reduce, join, window 등) filesystem 이나 database 에 저장을 하도록 해준다.
- Spark Streaming 을 Machine learning 이나 graph processing 에 적용할 수도 있다.
    - machine learning: [https://spark.apache.org/docs/latest/ml-guide.html](https://spark.apache.org/docs/latest/ml-guide.html)
    - Graph processing: [https://spark.apache.org/docs/latest/graphx-programming-guide.html](https://spark.apache.org/docs/latest/graphx-programming-guide.html)
- 들어오는 스트림 데이터는 배치형식으로 한번에 처리할 수 있게끔 나눠서 처리를 한다.
-

## **A Quick Example**

Before we go into the details of how to write your own Spark Streaming program, let’s take a quick look at what a simple Spark Streaming program looks like. Let’s say we want to count the number of words in text data received from a data server listening on a TCP socket. All you need to do is as follows.

```scala
import org.apache.spark._
import org.apache.spark.streaming._
import org.apache.spark.streaming.StreamingContext._ // not necessary since Spark 1.3

// Create a local StreamingContext with two working thread and batch interval of 1 second.
// The master requires 2 cores to prevent a starvation scenario.

val conf = new SparkConf().setMaster("local[2]").setAppName("NetworkWordCount")
val ssc = new StreamingContext(conf, Seconds(1))
```

Using this context, we can create a DStream that represents streaming data from a TCP source, specified as hostname (e.g. `localhost`) and port (e.g. `9999`).

```scala
// Create a DStream that will connect to hostname:port, like localhost:9999
val lines = ssc.socketTextStream("localhost", 9999)
```

This `lines` DStream represents the stream of data that will be received from the data server. Each record in this DStream is a line of text. Next, we want to split the lines by space characters into words.

```scala
// Split each line into words
val words = lines.flatMap(_.split(" "))
```

`flatMap` is a one-to-many DStream operation that creates a new DStream by generating multiple new records from each record in the source DStream. In this case, each line will be split into multiple words and the stream of words is represented as the `words` DStream. Next, we want to count these words.

```scala
import org.apache.spark.streaming.StreamingContext._ // not necessary since Spark 1.3
// Count each word in each batch
val pairs = words.map(word => (word, 1))
val wordCounts = pairs.reduceByKey(_ + _)

// Print the first ten elements of each RDD generated in this DStream to the console
wordCounts.print()
```

The `words` DStream is further mapped (one-to-one transformation) to a DStream of `(word, 1)` pairs, which is then reduced to get the frequency of words in each batch of data. Finally, `wordCounts.print()` will print a few of the counts generated every second.

Note that when these lines are executed, Spark Streaming only sets up the computation it will perform when it is started, and no real processing has started yet. To start the processing after all the transformations have been setup, we finally call

```scala
ssc.start()             // Start the computation
ssc.awaitTermination()  // Wait for the computation to terminate
```

The complete code can be found in the Spark Streaming example [NetworkWordCount](https://github.com/apache/spark/blob/v3.2.0/examples/src/main/scala/org/apache/spark/examples/streaming/NetworkWordCount.scala).

If you have already [downloaded](https://spark.apache.org/docs/latest/index.html#downloading) and [built](https://spark.apache.org/docs/latest/index.html#building) Spark, you can run this example as follows. You will first need to run Netcat (a small utility found in most Unix-like systems) as a data server by using

```scala
$ nc -lk 9999
```

Then, in a different terminal, you can start the example by using

- 여기서 나오는 예제는 Simple Spark Streaming 이다.
- 여기서 정의하는 DStream 은 시간별로 들어오는 데이터들의 연속적인 집합을 나타낸다.
- flatMap 은 one-to-many 연산이다.
- Spark 에서 프로그래밍 할 땐 데이터가 다들 Executor 에게 흩어져 있으니까, 이를 생각하고 프로그래밍해야한다. 병렬적으로 흩어져 있으니까 한 군데로 모우기 전에 최대한 처리를 해주고 모우는 식으로.
- StreamingContext 를 시작하지 않으면, start() 메소드를 호출하지 않으면 시작하지 않는다.
- StreamingContext.awaitTermination() 를 통해서 중지 신호를 받으면 중지할 수 있도록 해줘야한다.

---

## **Basic Concepts**

Next, we move beyond the simple example and elaborate on the basics of Spark Streaming.

### **Linking**

Similar to Spark, Spark Streaming is available through Maven Central. To write your own Spark Streaming program, you will have to add the following dependency to your SBT or Maven project.

```scala
// sbt Example
libraryDependencies += "org.apache.spark" % "spark-streaming_2.12" % "3.2.0" % "provided"
```

For ingesting data from sources like Kafka and Kinesis that are not present in the Spark Streaming core API, you will have to add the corresponding artifact `spark-streaming-xyz_2.12` to the dependencies. For example, some of the common ones are as follows.

```scala
Kafka - spark-streaming-kafka-0-10_2.12
Kinesis - spark-streaming-kinesis-asl_2.12 [Amazon Software License]
```

For an up-to-date list, please refer to the [Maven repository](https://search.maven.org/#search%7Cga%7C1%7Cg%3A%22org.apache.spark%22%20AND%20v%3A%223.2.0%22) for the full list of supported sources and artifacts.

---

### **Initializing StreamingContext**

To initialize a Spark Streaming program, a **StreamingContext** object has to be created which is the main entry point of all Spark Streaming functionality.

```scala
import org.apache.spark._
import org.apache.spark.streaming._

val conf = new SparkConf().setAppName(appName).setMaster(master)
val ssc = new StreamingContext(conf, Seconds(1))
```

The `appName` parameter is a name for your application to show on the cluster UI. `master` is a [Spark, Mesos, Kubernetes or YARN cluster URL](https://spark.apache.org/docs/latest/submitting-applications.html#master-urls), or a special **“local[*]”** string to run in local mode. In practice, when running on a cluster, you will not want to hardcode `master` in the program, but rather [launch the application with `spark-submit`](https://spark.apache.org/docs/latest/submitting-applications.html) and receive it there. However, for local testing and unit tests, you can pass “local[*]” to run Spark Streaming in-process (detects the number of cores in the local system). Note that this internally creates a [SparkContext](https://spark.apache.org/docs/latest/api/scala/org/apache/spark/SparkContext.html) (starting point of all Spark functionality) which can be accessed as `ssc.sparkContext`.

The batch interval must be set based on the latency requirements of your application and available cluster resources. See the [Performance Tuning](https://spark.apache.org/docs/latest/streaming-programming-guide.html#setting-the-right-batch-interval) section for more details.

A `StreamingContext` object can also be created from an existing `SparkContext` object.

```scala
import org.apache.spark.streaming._

val sc = ...                // existing SparkContext
val ssc = new StreamingContext(sc, Seconds(1))
```

After a context is defined, you have to do the following.

1. Define the input sources by creating input DStreams.
2. Define the streaming computations by applying transformation and output operations to DStreams.
3. Start receiving data and processing it using `streamingContext.start()`.
4. Wait for the processing to be stopped (manually or due to any error) using `streamingContext.awaitTermination()`.
5. The processing can be manually stopped using `streamingContext.stop()`.

### **Points to remember:**

Once a context has been started, no new streaming computations can be set up or added to it.

Once a context has been stopped, it cannot be restarted.

Only one StreamingContext can be active in a JVM at the same time.

stop() on StreamingContext also stops the SparkContext. To stop only the StreamingContext, set the optional parameter of `stop()` called `stopSparkContext` to false.

A SparkContext can be re-used to create multiple StreamingContexts, as long as the previous StreamingContext is stopped (without stopping the SparkContext) before the next StreamingContext is created.

- StreamingContext 오브젝트를 만들어야지 Spark Streaming Program 을 실행하는게 가능하다. 여기서 부터 main entry point 가 된다.
- StreamingContext 는 SparkConf 라는 설정정보 객체를 통해서 만들어진다.
    - 여기서 설정한 appName 은 cluster UI 에서 보일 앱의 이름이다.
    - master 는 이 분산된 클러스터가 실행될 URL 을 명시한다. 로컬모드, Standalne 모드, Mesos 모드, Yarn, Kubernetes 등의 url 을 넣을 수 있다. 주의할 건 여기에다가 하드코딩 하지는 말자.
- batch interval 은 latency 와 사용가능한 리소스를 보고 정하자. (Performance Detail 에서 자세히 다룸.)
- streamingContext.stop() 을 통해서 수동으로 멈추는 것도 가능하다.
- 기억해야될 요소는 다음과 같다.
    - Context 가 한번 시작되고 나면 새로운 스트리밍 계산을 시작하거나 추가할 수 없다.
    - Context 가 멈추면 여기서 다시 재시작할 수 없다. (기다려주지 않는다는 뜻인듯.)
    - JVM 에서 하나의  Context 만 작동할 수 있다.

---

## **Discretized Streams (DStreams)**

**Discretized Stream** or **DStream** is the basic abstraction provided by Spark Streaming. It represents a continuous stream of data, either the input data stream received from source, or the processed data stream generated by transforming the input stream. Internally, a DStream is represented by a continuous series of RDDs, which is Spark’s abstraction of an immutable, distributed dataset (see [Spark Programming Guide](https://spark.apache.org/docs/latest/rdd-programming-guide.html#resilient-distributed-datasets-rdds) for more details). Each RDD in a DStream contains data from a certain interval, as shown in the following figure.

![Untitled](../images/spark_streaming_guide(4).png)

Any operation applied on a DStream translates to operations on the underlying RDDs. For example, in the [earlier example](https://spark.apache.org/docs/latest/streaming-programming-guide.html#a-quick-example) of converting a stream of lines to words, the `flatMap` operation is applied on each RDD in the `lines` DStream to generate the RDDs of the `words` DStream. This is shown in the following figure.

These underlying RDD transformations are computed by the Spark engine. The DStream operations hide most of these details and provide the developer with a higher-level API for convenience. These operations are discussed in detail in later sections.

- Discretization 은 이산화라는 뜻으로 응용수학에서 연속적인 함수, 모델, 변수, 방정식을 이산적인 구성요소로 변환하는 프로세스를 말한다. (이산이란 뜻은 연속과 반대로 서로 떨어져 있다는 개념이다.)
- DStream 은 데이터의 스트림을 추상화한 개념이라고 생각하면 된다. 들어오는 데이터부터 변환되서 나가는데이터까지.
- 그리고 내부적으로 DStream 은 RDD 들의 집합으로 표현된다. DStream 에서의 연산은 RDD 의 연산으로 표현되고 이건 자바나 스칼라의 객체들 (= 데이터들) 이라고 생각하면 된다.
- RDD (Resilient Distriubted Dataset) 에 대해서 TMI 처럼 조금 설명하자면 RDD 는 스파크의 기본 데이터 구조를 말한다. 스파크의 모든 작업은 새로운 RDD 를 만들거나 존재하는 RDD 를 변형하거나 결과 계산을 위해 RDD 에서 연산하는 것을 말한다.
- 스파크는 map-reduce 작업을 RDD 를 통해서 해결한다고 하는데 이게 하둡에서 적용하는 map-reduce 와 차이점이 뭔지 잠깐 보자.
- 하둡의 map reduce 는 많이 느리다. 그런게 데이터 복제, 직렬화, 역직렬화 디스크 IO 로 인한 오버헤드가 중간 계산마다 있다. 실제로 이 작엄만을 수행하는데 90 % 이상 쓴다고 한다.
- 하지만 스파크의 RDD 는 메모리 위에서 이 작업들이 이뤄진다. 갱신의 문제는 RDD 에서 데이터는 read-only 로만 사용이 되기 때문에 신경쓰지 않아도 된다. 그래서 fault-tolearnt 하게 사용이 가능하다.

## **Input DStreams and Receivers**

Input DStreams are DStreams representing the stream of input data received from streaming sources. In the [quick example](https://spark.apache.org/docs/latest/streaming-programming-guide.html#a-quick-example), `lines` was an input DStream as it represented the stream of data received from the netcat server. Every input DStream (except file stream, discussed later in this section) is associated with a **Receiver** ([Scala doc](https://spark.apache.org/docs/latest/api/scala/org/apache/spark/streaming/receiver/Receiver.html), [Java doc](https://spark.apache.org/docs/latest/api/java/org/apache/spark/streaming/receiver/Receiver.html)) object which receives the data from a source and stores it in Spark’s memory for processing.

Spark Streaming provides two categories of built-in streaming sources.

- *Basic sources*: Sources directly available in the StreamingContext API. Examples: file systems, and socket connections.
- *Advanced sources*: Sources like Kafka, Kinesis, etc. are available through extra utility classes. These require linking against extra dependencies as discussed in the [linking](https://spark.apache.org/docs/latest/streaming-programming-guide.html#linking) section.

We are going to discuss some of the sources present in each category later in this section.

Note that, if you want to receive multiple streams of data in parallel in your streaming application, you can create multiple input DStreams (discussed further in the [Performance Tuning](https://spark.apache.org/docs/latest/streaming-programming-guide.html#level-of-parallelism-in-data-receiving) section). This will create multiple receivers which will simultaneously receive multiple data streams. But note that a Spark worker/executor is a long-running task, hence it occupies one of the cores allocated to the Spark Streaming application. Therefore, it is important to remember that a Spark Streaming application needs to be allocated enough cores (or threads, if running locally) to process the received data, as well as to run the receiver(s).

### **Basic Sources**

We have already taken a look at the `ssc.socketTextStream(...)` in the [quick example](https://spark.apache.org/docs/latest/streaming-programming-guide.html#a-quick-example) which creates a DStream from text data received over a TCP socket connection. Besides sockets, the StreamingContext API provides methods for creating DStreams from files as input sources.

### **File Streams**

For reading data from files on any file system compatible with the HDFS API (that is, HDFS, S3, NFS, etc.), a DStream can be created as via `StreamingContext.fileStream[KeyClass, ValueClass, InputFormatClass]`.

File streams do not require running a receiver so there is no need to allocate any cores for receiving file data.

For simple text files, the easiest method is `StreamingContext.textFileStream(dataDirectory)`.

### **How Directories are Monitored**

Spark Streaming will monitor the directory `dataDirectory` and process any files created in that directory.

- A simple directory can be monitored, such as `"hdfs://namenode:8040/logs/"`. All files directly under such a path will be processed as they are discovered.
- A [POSIX glob pattern](http://pubs.opengroup.org/onlinepubs/009695399/utilities/xcu_chap02.html#tag_02_13_02) can be supplied, such as `"hdfs://namenode:8040/logs/2017/*"`. Here, the DStream will consist of all files in the directories matching the pattern. That is: it is a pattern of directories, not of files in directories.
- All files must be in the same data format.
- A file is considered part of a time period based on its modification time, not its creation time.
- Once processed, changes to a file within the current window will not cause the file to be reread. That is: *updates are ignored*.
- The more files under a directory, the longer it will take to scan for changes — even if no files have been modified.
- If a wildcard is used to identify directories, such as `"hdfs://namenode:8040/logs/2016-*"`, renaming an entire directory to match the path will add the directory to the list of monitored directories. Only the files in the directory whose modification time is within the current window will be included in the stream.
- Calling `[FileSystem.setTimes()](https://hadoop.apache.org/docs/current/api/org/apache/hadoop/fs/FileSystem.html#setTimes-org.apache.hadoop.fs.Path-long-long-)` to fix the timestamp is a way to have the file picked up in a later window, even if its contents have not changed.

### **Using Object Stores as a source of data**

“Full” Filesystems such as HDFS tend to set the modification time on their files as soon as the output stream is created. When a file is opened, even before data has been completely written, it may be included in the `DStream` - after which updates to the file within the same window will be ignored. That is: changes may be missed, and data omitted from the stream.

To guarantee that changes are picked up in a window, write the file to an unmonitored directory, then, immediately after the output stream is closed, rename it into the destination directory. Provided the renamed file appears in the scanned destination directory during the window of its creation, the new data will be picked up.

In contrast, Object Stores such as Amazon S3 and Azure Storage usually have slow rename operations, as the data is actually copied. Furthermore, renamed object may have the time of the `rename()` operation as its modification time, so may not be considered part of the window which the original create time implied they were.

Careful testing is needed against the target object store to verify that the timestamp behavior of the store is consistent with that expected by Spark Streaming. It may be that writing directly into a destination directory is the appropriate strategy for streaming data via the chosen object store.

For more details on this topic, consult the [Hadoop Filesystem Specification](https://hadoop.apache.org/docs/stable2/hadoop-project-dist/hadoop-common/filesystem/introduction.html).

- input DStream 은 Stream Source 로 부터 데이터를 받는 스트림을 말한다.
- 그리고 Receiver 는 Input DStream 의 데이터를 받아서 스파크 메모리에 올리는 역할을 한다.
- Spark Streaming 은 두 가지 종류의 Streaming Source 를 가진다.
    - Basic sources: StreamingContext API 에서 직접적으로 사용이 가능한 것. FileSystem 이나 Socket connection 같은 것들.
    - Advanced sources: Kafka 나 Kinesis 같은 것. 외부 유틸리티 클래스를 통해서 접근이 가능한 것. 이 것들은 외부 의존성들이 필요하다.
- 만약에 Multiple Input Stream 을 가진다면 그만큼 스레드가 생기고 CPU 코어가 필요하다는 사실을 알고있자.
- 즉 이 말을 이해한다면 로컬 모드로 실행할 때 local 이나 local[1] 을 하지는 않을 것이다. Receiver 가 하나의 스레드에서 동작하고 있을 것이므로 실제로 데이터를 처리하는 스레드가 없기 때문에. 그러므로 n 을 리시버의 개수보다 많도록 하자.
- 파일 시스템을 이용할 경우 (예, HDFS, S3, NFS) 리시버를 유지할 필요는 없다.
- Basic Source 에 대한 설명
    - Spark Streaming 은 dataDirectory 에 관한 모니터링을 할 수 있고 여기에 있는 파일들을 처리하는게 가능하다.
    - 여기에 있는 파일들은 모두 같은 포맷을 가져야한다.
    - 파일들은 생성된 시간이 아니라 수정된 시간에 따라서 분류된다.
    - 일단 한번 파일이 처리되면 현재의 Window 에서 파일을 다시 읽지는 않는다. 즉 처리된 파일에서 추가적인 변경을 하면 이 변경은 유실될 수 있다.
        - 여기서 Window 는 해당 시간동안 모인 데이터들을 말한다. RDD 에 해당하는 데이터들로
    - 디렉토리에 파일이 많을수록 변경이 된 파일들을 찾는데 오랜 시간이 걸린다. (변경이 없더라도.)
    - FileSystem.setTImes() 를 통해 타임스탬프를 바꿔서 해당 타임때의 window 를 가져오는 것도 가능하다.
    - 하둡의 파일 시스템의 경우 Output Stream 이 만들어지자마자 파일의 변경 시간을 기록한다. 이 말은 파일이 완전히 기록되기도 전에 파일이 열리면 윈도우 내의 파일 변경이 무시될 수 있다는 것이다.
    - 이 변경사항을 적용하려면 모니터링 되지 않는 디렉토리 (= no Destination directory) 에 파일을 쓴 다음에 출력 스트림이 닫힌 직후 즉 시 디렉토리 이름을 Desination Directory 로 바꿔라. Window 가 생성되는 시점에
        - 하둡의 파일 시스템은 쓰는 중에 읽는게 가능하다는 거 같은데.
        - Hadoop 에서 operation 의 atomic 은 rename() 연산과 delete() 연산, create() 연산에서만 보장된다.
        - 원래는 outputStream 이 close() 될 때 modification time 이 등록되어야 하는 거 같다.
        - 하둡의 Concurrency 는 isolation 완전 보장을 하지는 않는다.
    - Kafka 와 Kinesis 를 Source 로 사용할 땐 스파크 이외의 라이브러리가 필요하다. 그래서 의존성을 추가해야하는데 의존성 충돌이 나지 않도록 주의하자.

## **Transformations on DStreams**

Similar to that of RDDs, transformations allow the data from the input DStream to be modified. DStreams support many of the transformations available on normal Spark RDD’s. Some of the common ones are as follows.

[Transforamtion](https://www.notion.so/98fcade93e264b4f8e5b2ffb7a8bd0bf)

- countByValue: Value 를 가지고 개수를 세는데 이때 value 가 Key 가 된다. return new DStream of (K, Long) pair
- reduceByKey: (K,V) Pair 에서 Key 를 기반으로 집계를 하는데 Value 를 가지고 reduce() 연산을 하는 경우. 그리고 reduceByKey 의 grouping 의 경우 local 모두에서는 2. 클러스터 모드에서는 `spark.default.parallelism` 의 값에 따라서 결정된다.
    - groupByKey 대신에 reduceByKey 를 사용하자. 셔플링을 하기전에 자신의 파티션에서 데이터를 combine 을 먼저 수행하기 때문이다. 스파크는 하나의 Executor 가 가진 메모리보다 더 많은 셔플링을 하는 경우에 데이터를 디스크에 저장해놓고 사용한다. 그리고 하나의 키로 집계했는데 키-값 데이터가 executor 메모리를 넘어가는 경우에 OutOfMemory 에러를 낸다. (셔플을 하는 경우는 데이터를 나눠서 전송할 수 있으니까 디스크 I/O 를 쓰는 반면에 실제 연산을 하기 위해서는 모운 데이터를 모두 메모리에 올려야하니까 이 경우에는 OutofMemory 에러를 내네.)
- join: (K,V) 와 (K,W) pair 의 DStream 에서 조인을 홏출하면 (K, (V,W)) Pair 가 리턴된다.
- cogroup
- transform
- updateStateByKey

### **Transform Operation**

The `transform` operation (along with its variations like `transformWith`) allows arbitrary RDD-to-RDD functions to be applied on a DStream. It can be used to apply any RDD operation that is not exposed in the DStream API. For example, the functionality of joining every batch in a data stream with another dataset is not directly exposed in the DStream API. However, you can easily use `transform` to do this. This enables very powerful possibilities. For example, one can do real-time data cleaning by joining the input data stream with precomputed spam information (maybe generated with Spark as well) and then filtering based on it.

- transform operation 은 DStream (= RDD 의 연속) 에서만 적용이 가능할 뿐 아니라 DStream API 가 아닌 RDD 에서도 충분히 사용가능하다. 예를 들면 실시간 데이터를 정리하는 작업을 할 때 미리 계산된 스팸 정보와 조인해서 필터링 작업을 수행하는게 가능하다.
- 즉 DStream 에 있는 RDD 뿐 아니라 다른 RDD 와도 결합해서 사용이 가능하다는 것 같다.

### **Window Operations**

Spark Streaming also provides *windowed computations*, which allow you to apply transformations over a sliding window of data. The following figure illustrates this sliding window.

![스크린샷 2022-01-30 오전 2.21.24.png](../images/spark_streaming_guide(4).png)

As shown in the figure, every time the window *slides* over a source DStream, the source RDDs that fall within the window are combined and operated upon to produce the RDDs of the windowed DStream. In this specific case, the operation is applied over the last 3 time units of data, and slides by 2 time units. This shows that any window operation needs to specify two parameters.

- *window length* - The duration of the window (3 in the figure).
- *sliding interval* - The interval at which the window operation is performed (2 in the figure).

These two parameters must be multiples of the batch interval of the source DStream (1 in the figure).

Let’s illustrate the window operations with an example. Say, you want to extend the [earlier example](https://spark.apache.org/docs/latest/streaming-programming-guide.html#a-quick-example) by generating word counts over the last 30 seconds of data, every 10 seconds. To do this, we have to apply the `reduceByKey` operation on the `pairs` DStream of `(word, 1)` pairs over the last 30 seconds of data. This is done using the operation `reduceByKeyAndWindow`.

[Window Opeartion](https://www.notion.so/0c7d13c197bf4ed29b606ad669c8a439)

- spark streaming 에서는 window Operation 도 지원한다.
- Sliding Window 의 형태는 위의 그림과 같은데 window length 와 sliding interval 이라는 두 주요 특성으로 구별된다.

## **Join Operations**

Finally, its worth highlighting how easily you can perform different kinds of joins in Spark Streaming.

### **Stream-stream joins**

Streams can be very easily joined with other streams.

```scala
val stream1: DStream[String, String] = ...
val stream2: DStream[String, String] = ...
val joinedStream = stream1.join(stream2)
```

Here, in each batch interval, the RDD generated by `stream1` will be joined with the RDD generated by `stream2`. You can also do `leftOuterJoin`, `rightOuterJoin`, `fullOuterJoin`. Furthermore, it is often very useful to do joins over windows of the streams. That is pretty easy as well.

```scala
val windowedStream1 = stream1.window(Seconds(20))
val windowedStream2 = stream2.window(Minutes(1))
val joinedStream = windowedStream1.join(windowedStream2)
```

- Spark Streaming 에서 조인을 얼마나 쉽게 쓸 수 있는지 다양한 예를 통해서 소개시켜주는 것.
- 다른 Stream 에 있는 rdd 끼리, window 끼리 조인이 가능하다.

### **Stream-dataset joins**

This has already been shown earlier while explain `DStream.transform` operation. Here is yet another example of joining a windowed stream with a dataset.

```scala
val dataset: RDD[String, String] = ...
val windowedStream = stream.window(Seconds(20))...
val joinedStream = windowedStream.transform { rdd => rdd.join(dataset) }
```

In fact, you can also dynamically change the dataset you want to join against. The function provided to `transform` is evaluated every batch interval and therefore will use the current dataset that `dataset` reference points to.

The complete list of DStream transformations is available in the API documentation. For the Scala API, see [DStream](https://spark.apache.org/docs/latest/api/scala/org/apache/spark/streaming/dstream/DStream.html) and [PairDStreamFunctions](https://spark.apache.org/docs/latest/api/scala/org/apache/spark/streaming/dstream/PairDStreamFunctions.html). For the Java API, see [JavaDStream](https://spark.apache.org/docs/latest/api/java/index.html?org/apache/spark/streaming/api/java/JavaDStream.html) and [JavaPairDStream](https://spark.apache.org/docs/latest/api/java/index.html?org/apache/spark/streaming/api/java/JavaPairDStream.html). For the Python API, see [DStream](https://spark.apache.org/docs/latest/api/python/reference/api/pyspark.streaming.DStream.html#pyspark.streaming.DStream).

- 데이터셋의 경우 원하는 타입으로 언제든지 런타임 시점에 변경할 수 있다. 그리고 이를 통해서 조인을 하면 된다.

## **Output Operations on DStreams**

Output operations allow DStream’s data to be pushed out to external systems like a database or a file systems. Since the output operations actually allow the transformed data to be consumed by external systems, they trigger the actual execution of all the DStream transformations (similar to actions for RDDs). Currently, the following output operations are defined:

[Output Opeartion ](https://www.notion.so/7994af1ba49f4507b8f89edc10c4a573)

- Output Operation 은 DStream 의 처리된 데이터를 외부 시스템으로 저장하도록 할 수 있다. 즉 DStream 의 transformation 의 트리거 역할을 해준다.
- 여기서 알아야 하는 연산은 `foreachRDD(func)` 인데 이는 각각의 RDD 를 주어진 파라미터인 function 을 통해서 실행한다. 중요한 건 이 실행은 Driver Process 에 의해서 실행된다는 점이다.

### **Design Patterns for using foreachRDD**

`dstream.foreachRDD` is a powerful primitive that allows data to be sent out to external systems. However, it is important to understand how to use this primitive correctly and efficiently. Some of the common mistakes to avoid are as follows.

Often writing data to external system requires creating a connection object (e.g. TCP connection to a remote server) and using it to send data to a remote system. For this purpose, a developer may inadvertently try creating a connection object at the Spark driver, and then try to use it in a Spark worker to save records in the RDDs. For example (in Scala),

```scala
dstream.foreachRDD { rdd =>
  val connection = createNewConnection()  // executed at the driver
  rdd.foreach { record =>
    connection.send(record) // executed at the worker
  }
}
```

This is incorrect as this requires the connection object to be serialized and sent from the driver to the worker. Such connection objects are rarely transferable across machines. This error may manifest as serialization errors (connection object not serializable), initialization errors (connection object needs to be initialized at the workers), etc. The correct solution is to create the connection object at the worker.

However, this can lead to another common mistake - creating a new connection for every record. For example,

```scala
dstream.foreachRDD { rdd =>
  rdd.foreach { record =>
    val connection = createNewConnection()
    connection.send(record)
    connection.close()
  }
}
```

Typically, creating a connection object has time and resource overheads. Therefore, creating and destroying a connection object for each record can incur unnecessarily high overheads and can significantly reduce the overall throughput of the system. A better solution is to use `rdd.foreachPartition` - create a single connection object and send all the records in a RDD partition using that connection.

```scala
dstream.foreachRDD { rdd =>
  rdd.foreachPartition { partitionOfRecords =>
    val connection = createNewConnection()
    partitionOfRecords.foreach(record => connection.send(record))
    connection.close()
  }
}
```

This amortizes the connection creation overheads over many records.

Finally, this can be further optimized by reusing connection objects across multiple RDDs/batches. One can maintain a static pool of connection objects than can be reused as RDDs of multiple batches are pushed to the external system, thus further reducing the overheads.

```scala
dstream.foreachRDD { rdd =>
  rdd.foreachPartition { partitionOfRecords =>
    // ConnectionPool is a static, lazily initialized pool of connections
    val connection = ConnectionPool.getConnection()
    partitionOfRecords.foreach(record => connection.send(record))
    ConnectionPool.returnConnection(connection)  // return to the pool for future reuse
  }
}
```

Note that the connections in the pool should be lazily created on demand and timed out if not used for a while. This achieves the most efficient sending of data to external systems.

### **Other points to remember:**

DStreams are executed lazily by the output operations, just like RDDs are lazily executed by RDD actions. Specifically, RDD actions inside the DStream output operations force the processing of the received data. Hence, if your application does not have any output operation, or has output operations like `dstream.foreachRDD()` without any RDD action inside them, then nothing will get executed. The system will simply receive the data and discard it.

By default, output operations are executed one-at-a-time. And they are executed in the order they are defined in the application.

- 그냥 foreachRDD 를 사용할 때 조심해야 할 사항에 대해서 알려주는 내용.
- foreachRDD 를 사용할 때는 외부 시스템에 RDD 를 저장하려는 용도로 많이 쓸텐데 이때는 외부 시스템과 연결할 Connection 객체가 필요할 것이다. 근데 foreachRDD 내부에서 connection 객체를 만들면 이는 드라이버 프로세스에서 만들어지는데 실제로 저장을 처리하는 로직은 Worker Node 에서 필요하니까 직렬화해서 보내야한다. 근데 Connection 객체는 직렬화 되지 않으니까 에러가 생길 것.
- 그렇다고해서 rdd 를 프로세싱 하는 로직에서 커넥션 객체를 만들면 매번 레코드마다 만들고 폐기하는건 오버헤드가 심하다. 그러니까 파티션별로 커넥션 객체를 만들고 더 나아가서 풀로 활용하도록 하는 걸 권장한다. rdd.foreachPartition 을 통해서 파티션별로 한번만 처리해야하는 로직을 넣어서 이를 처리하도록 하자.

## **DataFrame and SQL Operations**

You can easily use [DataFrames and SQL](https://spark.apache.org/docs/latest/sql-programming-guide.html) operations on streaming data. You have to create a SparkSession using the SparkContext that the StreamingContext is using. Furthermore, this has to done such that it can be restarted on driver failures. This is done by creating a lazily instantiated singleton instance of SparkSession. This is shown in the following example. It modifies the earlier [word count example](https://spark.apache.org/docs/latest/streaming-programming-guide.html#a-quick-example) to generate word counts using DataFrames and SQL. Each RDD is converted to a DataFrame, registered as a temporary table and then queried using SQL.

```scala
val words: DStream[String] = ...

words.foreachRDD { rdd =>

  // Get the singleton instance of SparkSession
  val spark = SparkSession.builder.config(rdd.sparkContext.getConf).getOrCreate()
  import spark.implicits._

  // Convert RDD[String] to DataFrame
  val wordsDataFrame = rdd.toDF("word")

  // Create a temporary view
  wordsDataFrame.createOrReplaceTempView("words")

  // Do word count on DataFrame using SQL and print it
  val wordCountsDataFrame = 
    spark.sql("select word, count(*) as total from words group by word")
  wordCountsDataFrame.show()
}
```

See the full [source code](https://github.com/apache/spark/blob/v3.2.0/examples/src/main/scala/org/apache/spark/examples/streaming/SqlNetworkWordCount.scala).

You can also run SQL queries on tables defined on streaming data from a different thread (that is, asynchronous to the running StreamingContext). Just make sure that you set the StreamingContext to remember a sufficient amount of streaming data such that the query can run. Otherwise the StreamingContext, which is unaware of the any asynchronous SQL queries, will delete off old streaming data before the query can complete. For example, if you want to query the last batch, but your query can take 5 minutes to run, then call `streamingContext.remember(Minutes(5))` (in Scala, or equivalent in other languages).

See the [DataFrames and SQL](https://spark.apache.org/docs/latest/sql-programming-guide.html) guide to learn more about DataFrames.

- DataFrame 과 SQL 을 Streaming Data 에 적용하는게 가능하다. 이를 사용할려면 SparkSession 을 만들어야 한다. 이는 SparkContext 가 필요하며 SparkContext 는 StreamingContext 에 있다.
- SQL 을 쓸 때 조심해야할 사항들에서도 이야기하고 있는데 SQL 쿼리는 테이블에서 이뤄진다. Spark Streaming 에서는 Streaming Data 가 테이블 역할을 할 것이다. 그러므로 쿼리가 실행되는데 5 분정도 걸린다면 Steaming 이 이 데이터들을 기억하지 못할수도 있다. 즉 오래걸리는 작업이 있다면 이를 기억하도록 streamingContext.remember() 를 호출하자.

## **Caching / Persistence**

Similar to RDDs, DStreams also allow developers to persist the stream’s data in memory. That is, using the `persist()` method on a DStream will automatically persist every RDD of that DStream in memory. This is useful if the data in the DStream will be computed multiple times (e.g., multiple operations on the same data). For window-based operations like `reduceByWindow` and `reduceByKeyAndWindow` and state-based operations like `updateStateByKey`, this is implicitly true. Hence, DStreams generated by window-based operations are automatically persisted in memory, without the developer calling `persist()`.

For input streams that receive data over the network (such as, Kafka, sockets, etc.), the default persistence level is set to replicate the data to two nodes for fault-tolerance.

Note that, unlike RDDs, the default persistence level of DStreams keeps the data serialized in memory. This is further discussed in the [Performance Tuning](https://spark.apache.org/docs/latest/streaming-programming-guide.html#memory-tuning) section. More information on different persistence levels can be found in the [Spark Programming Guide](https://spark.apache.org/docs/latest/rdd-programming-guide.html#rdd-persistence).

- RDD 와 유사하게 DStream 도 데이터를 메모리에 저장하는게 가능하다. persist() 메소드를 호출한다면.
- 울론 DStream 에서 여러번 계산해야하는 데이터들, 메모리에 저장해야 효율적인 연산들 like `reduceByWindow` , `reduceByKeyAndWindow` , `updateStateByKey` 같은 메소드는 persist() 메소드를 호출하지 않아도 자동으로 메모리에 저장을 해둔다.
- 메모리에 저장해두는 레벨은 memory-serializer 이다.

## **Deploying Applications**

This section discusses the steps to deploy a Spark Streaming application.

### **Requirements**

To run a Spark Streaming applications, you need to have the following.

- *Cluster with a cluster manager* - This is the general requirement of any Spark application, and discussed in detail in the [deployment guide](https://spark.apache.org/docs/latest/cluster-overview.html).
- *Package the application JAR* - You have to compile your streaming application into a JAR. If you are using `[spark-submit](https://spark.apache.org/docs/latest/submitting-applications.html)` to start the application, then you will not need to provide Spark and Spark Streaming in the JAR. However, if your application uses [advanced sources](https://spark.apache.org/docs/latest/streaming-programming-guide.html#advanced-sources) (e.g. Kafka), then you will have to package the extra artifact they link to, along with their dependencies, in the JAR that is used to deploy the application. For example, an application using `KafkaUtils` will have to include `spark-streaming-kafka-0-10_2.12` and all its transitive dependencies in the application JAR.
- *Configuring checkpointing* - If the stream application requires it, then a directory in the Hadoop API compatible fault-tolerant storage (e.g. HDFS, S3, etc.) must be configured as the checkpoint directory and the streaming application written in a way that checkpoint information can be used for failure recovery. See the [checkpointing](https://spark.apache.org/docs/latest/streaming-programming-guide.html#checkpointing) section for more details.

- spark-submit 을 이용해서 배포할려면 JAR 로 패키징 하지 않아도 되지만 Kafka 같은 Advanced source 를 이용하려면 의존성을 연결시키기 위해서 jar 로 패키징을 해야한다.
- stream application 이 checkpointing 이 필요하다면 이를 디렉토리로 설정하자. 이건 sparkcontext.checkpoint() 를 통해서 복구할 때 사용할 정보들을 저장하는 디렉토리로 사용할 수 있다.

## **Monitoring Applications**

## **Performance Tuning**

## **Setting the Right Batch Interval**

## **Memory Tuning**

Tuning the memory usage and GC behavior of Spark applications has been discussed in great detail in the [Tuning Guide](https://spark.apache.org/docs/latest/tuning.html#memory-tuning). It is strongly recommended that you read that. In this section, we discuss a few tuning parameters specifically in the context of Spark Streaming applications.

The amount of cluster memory required by a Spark Streaming application depends heavily on the type of transformations used. For example, if you want to use a window operation on the last 10 minutes of data, then your cluster should have sufficient memory to hold 10 minutes worth of data in memory. Or if you want to use `updateStateByKey` with a large number of keys, then the necessary memory will be high. On the contrary, if you want to do a simple map-filter-store operation, then the necessary memory will be low.

In general, since the data received through receivers is stored with StorageLevel.MEMORY_AND_DISK_SER_2, the data that does not fit in memory will spill over to the disk. This may reduce the performance of the streaming application, and hence it is advised to provide sufficient memory as required by your streaming application. Its best to try and see the memory usage on a small scale and estimate accordingly.

Another aspect of memory tuning is garbage collection. For a streaming application that requires low latency, it is undesirable to have large pauses caused by JVM Garbage Collection.

There are a few parameters that can help you tune the memory usage and GC overheads

- **Persistence Level of DStreams**: As mentioned earlier in the [Data Serialization](https://spark.apache.org/docs/latest/streaming-programming-guide.html#data-serialization) section, the input data and RDDs are by default persisted as serialized bytes. This reduces both the memory usage and GC overheads, compared to deserialized persistence. Enabling Kryo serialization further reduces serialized sizes and memory usage. Further reduction in memory usage can be achieved with compression (see the Spark configuration `spark.rdd.compress`), at the cost of CPU time.
- **Clearing old data**: By default, all input data and persisted RDDs generated by DStream transformations are automatically cleared. Spark Streaming decides when to clear the data based on the transformations that are used. For example, if you are using a window operation of 10 minutes, then Spark Streaming will keep around the last 10 minutes of data, and actively throw away older data. Data can be retained for a longer duration (e.g. interactively querying older data) by setting `streamingContext.remember`.

- 디테일한 튜닝은 Tuning Guide 를 따르자. (꼭 읽어보길 권하는듯.)
- 클러스터가 사용하는 메모리는 어떠한 연산을 하느냐에 따라 다르다. DStream 을 Window 형태로 10 분간의 데이터를 이용한다고 하거나 updateStateByKey 를 하는 경우에는 메모리가 많이 들것이지만 간단한 map-filter 연산만 한다고하면 메모리가 만히 필요하진 않을 것이다.
- 기본적으로 Receiver 가 데이터를 받아서 메모리에 올리는 전략은 Storage.MEMORY_AND_DISK_SER_2 이다. 직렬화를 하기 때문에 메모리를 좀 더 효율적으로 쓴다. 여기서 주의할 건 메모리가 넘쳐서 디스크에 쓰는 경우 직렬화 역직렬화 + I/O 작업 때문에 퍼포먼스가 안나올 수 있다. 그러므로 충분한 메모리를 제공해주거나 작은 양의 메모리로 충분하도록 변경할 필요가 있다.
- 또 다른 예로 GC 를 튜닝하는 경우는 어플리케이션에서 low latency 가 중요한 경우다. (Full GC 때문에 low latency 가 일어나지 않도록 하기 위해서)
- 기본적으로 모든 input data 와 transforming 한 RDD 데이터는 자동적으로 스파크 어플리케이션에서 지운다. 지울 때는 물론 이게 사용될 범위를 계산해서 더이상 필요없다고 하고 지우고 추가적으로 지우는 시간을 조금 더 기다려야하는 시간이 있다면 `streamingContext.remember` 메소드를 통해서 설정할 수 있다.
- GC 자체를 바꾸는 것도 좋다. CMS 나 G1 GC 를 쓰도록 하자.

## **Important points to remember:**

- A DStream is associated with a single receiver. For attaining read parallelism multiple receivers i.e. multiple DStreams need to be created. A receiver is run within an executor. It occupies one core. Ensure that there are enough cores for processing after re ceiver slots are booked i.e. `spark.cores.max` should take the receiver slots into account. The receivers are allocated to executors in a round robin fashion.
- When data is received from a stream source, receiver creates blocks of data. A new block of data is generated every blockInterval milliseconds. N blocks of data are created during the batchInterval where N = batchInterval/blockInterval. These blocks are distributed by the BlockManager of the current executor to the block managers of other executors. After that, the Network Input Tracker running on the driver is informed about the block locations for further processing.
- An RDD is created on the driver for the blocks created during the batchInterval. The blocks generated during the batchInterval are partitions of the RDD. Each partition is a task in spark. blockInterval== batchinterval would mean that a single partition is created and probably it is processed locally.
- Instead of relying on batchInterval and blockInterval, you can define the number of partitions by calling `inputDstream.repartition(n)`. This reshuffles the data in RDD randomly to create n number of partitions. Yes, for greater parallelism. Though comes at the cost of a shuffle. An RDD’s processing is scheduled by driver’s jobscheduler as a job. At a given point of time only one job is active. So, if one job is executing the other jobs are queued.
- If you have two dstreams there will be two RDDs formed and there will be two jobs created which will be scheduled one after the another. To avoid this, you can union two dstreams. This will ensure that a single unionRDD is formed for the two RDDs of the dstreams. This unionRDD is then considered as a single job. However, the partitioning of the RDDs is not impacted.
- If the batch processing time is more than batchinterval then obviously the receiver’s memory will start filling up and will end up in throwing exceptions (most probably BlockNotFoundException). Currently, there is no way to pause the receiver. Using SparkConf configuration `spark.streaming.receiver.maxRate`, rate of receiver can be limited.

- 하나의 receiver 가 하나의 DStream 을 처리하는 구조로 되어있고 하나의 CPU core 를 차지한다. 병렬성을 주고 싶다면 Multi Receiver 를 이용해야하며 여러개의 코어를 차지할 것이니 충분히 여유있게 코어를 Executor 에게 할당하자. Receiver 뿐만 아니라 실제로 처리하는 코어가 필요하니까.
- Receiver 는 DStream 을 받을 때 데이터를 블록 단위로 만든다. 물론 블록마다 만드는데 걸리는 시간인 Block Interval 이 있다. Block Interval 은 milliseconds 단위로 이뤄진다. (이러한 블록들이 모여서 데이터 그룹을 나타내는 배치를 만드는 것 같다.) 현재 Exeucutor 에 있는 BlockManager 를 통해서 이렇게 만들어진 블록들은 다른 Executor 의 BlockManager 에게 전달된다. 이 과정후에 이 정보들은 Driver 에서 실행중인 Network Input Tracker 에게 전달된다. 추가적인 처리를 위해서.
- RDD 는 외부 시스템 (하둡이나, S3) 같은 곳에서 만들어지거나 드라이버 프로세스에 의해서 만들어질 수 있다. RDD 를 이루는 파티션은 BatchInterval 동안 만들어진 블록들로 이뤄진다. (Executor 의 BlockManager 에 의해서 만들어진 블록들)
- DStream.repartition(n) 을 통해서 RDD 의 데이터를 랜덤적으로 섞어서 n 개의 파티션을 만들 수 있다. 이 과정에서 셔플은 발생한다. RDD Processing 은 드라이버의 JobScheduler 를 통해서 처리가 되며 하나의 잡으로서 다뤄진다. 하나의 잡이 실행중인 동안에 다른 잡들은 큐에서 대기한다.
- 두 개의 DStream 을 실행하면 두 개의 RDD 가 실행이되고 차례로 번갈아가면서 실행한다. 두 개의 DStream 을 하나로서 처리하고 싶다면 unionRDD 를 실행하면 된다.
- 배치 처리시간이 batchInterval 시간보다 길다면 메모리가 가득차서 BlockNotFoundException 이 발생할 수 있다. 이 경우에는 `spark.streaming.receiver.maxRate` 로 Receiver 의 속도를 제어하자.

## **Fault-tolerance Semantics**

In this section, we will discuss the behavior of Spark Streaming applications in the event of failures.

### **Background**

To understand the semantics provided by Spark Streaming, let us remember the basic fault-tolerance semantics of Spark’s RDDs.

1. An RDD is an immutable, deterministically re-computable, distributed dataset. Each RDD remembers the lineage of deterministic operations that were used on a fault-tolerant input dataset to create it.
2. If any partition of an RDD is lost due to a worker node failure, then that partition can be re-computed from the original fault-tolerant dataset using the lineage of operations.
3. Assuming that all of the RDD transformations are deterministic, the data in the final transformed RDD will always be the same irrespective of failures in the Spark cluster.

Spark operates on data in fault-tolerant file systems like HDFS or S3. Hence, all of the RDDs generated from the fault-tolerant data are also fault-tolerant. However, this is not the case for Spark Streaming as the data in most cases is received over the network (except when `fileStream` is used). To achieve the same fault-tolerance properties for all of the generated RDDs, the received data is replicated among multiple Spark executors in worker nodes in the cluster (default replication factor is 2). This leads to two kinds of data in the system that need to recovered in the event of failures:

1. *Data received and replicated* - This data survives failure of a single worker node as a copy of it exists on one of the other nodes.
2. *Data received but buffered for replication* - Since this is not replicated, the only way to recover this data is to get it again from the source.

Furthermore, there are two kinds of failures that we should be concerned about:

1. *Failure of a Worker Node* - Any of the worker nodes running executors can fail, and all in-memory data on those nodes will be lost. If any receivers were running on failed nodes, then their buffered data will be lost.
2. *Failure of the Driver Node* - If the driver node running the Spark Streaming application fails, then obviously the SparkContext is lost, and all executors with their in-memory data are lost.

With this basic knowledge, let us understand the fault-tolerance semantics of Spark Streaming.

- RDD 의 특징은 Immutable, deterministically re-computable, distributed dataset 이라는 점이다. Immutable 하다는 건 transforming 만 가능하지 직접적으로 변경이 안된다는 뜻이며 deterministic 하다는 점은 실패가나도 재연산을 통해셔 결국에는 같은 최종 데이터를 얻을 수 있다는 점이다. distributed dataset 은 병렬적으로 처리가 가능함을 말한다.
- determinisitic 의 특징은 fault-tolerant 한 저장소를 쓰고 있는 경우에는 당연하게 이 기능을 지원할 것이다. 하지만 Spark Streaming Application 의 경우에는 어떻게 이 기능을 지원받을 수 있을까? Receiver 가 데이터를 받으면 이걸 다른 Worker node 에 복제하기 때문에 가능하다. replication factor 는 기본적으로 2 이다.
- Spark Streaming 에서 효율적인 복제를 위해서 Buffer 를 쓰는데 버퍼에 데이터가 있지만 아직 복제되지는 않은 경우에 실패가나면 데이터가 유실될 수 있다. 이 경우에는 어쩔 수 없이 한번 더 받아야한다.
- 추가로 스파크에서는 두 종류의 실패가 있는데 이것도 알아보자. 워커노드가 실패한 경우에는 해당 워커노드의 버퍼에 있는 데이터만 잃어버릴 수 있지만 드라이버가 장애가 난 경우에는 모든 워커노드의 버퍼에 있는 데이터를 잃어버릴 수 있다.



### **Definitions**

The semantics of streaming systems are often captured in terms of how many times each record can be processed by the system. There are three types of guarantees that a system can provide under all possible operating conditions (despite failures, etc.)

1. *At most once*: Each record will be either processed once or not processed at all.
2. *At least once*: Each record will be processed one or more times. This is stronger than *at-most once* as it ensure that no data will be lost. But there may be duplicates.
3. *Exactly once*: Each record will be processed exactly once - no data will be lost and no data will be processed multiple times. This is obviously the strongest guarantee of the three.
- Streaming System 에서는 데이터를 몇번 처리하느냐에 따라서 구별될 수 있다.
- 정확하게 한 번, 최소 한 번, 최대 한 번.

### Basic Semantics

In any stream processing system, broadly speaking, there are three steps in processing the data.

1. *Receiving the data*: The data is received from sources using Receivers or otherwise.
2. *Transforming the data*: The received data is transformed using DStream and RDD transformations.
3. *Pushing out the data*: The final transformed data is pushed out to external systems like file systems, databases, dashboards, etc.

If a streaming application has to achieve end-to-end exactly-once guarantees, then each step has to provide an exactly-once guarantee. That is, each record must be received exactly once, transformed exactly once, and pushed to downstream systems exactly once. Let’s understand the semantics of these steps in the context of Spark Streaming.

1. *Receiving the data*: Different input sources provide different guarantees. This is discussed in detail in the next subsection.
2. *Transforming the data*: All data that has been received will be processed *exactly once*, thanks to the guarantees that RDDs provide. Even if there are failures, as long as the received input data is accessible, the final transformed RDDs will always have the same contents.
3. *Pushing out the data*: Output operations by default ensure *at-least once* semantics because it depends on the type of output operation (idempotent, or not) and the semantics of the downstream system (supports transactions or not). But users can implement their own transaction mechanisms to achieve *exactly-once* semantics. This is discussed in more details later in the section.
- Steaming 처리를 간략하게 구성한다면 이럴 것이다.
- 데이터를 가져와서 transforming 한 후 외부 시스템에 데이터를 pushing out 하는 것.
- 여기서 정확하게 한 번 처리를 하기 위해서는 Receiving the data, Transforming the data, Pushing out the data 모두 한번만 해야한다.
- Receiving Data 의 경우 어떤 Input Source 를 쓰는지에 따라서 다르다. 이는 뒤에서 하나씩 살펴보자.
- Transforming Data 의 경우 RDD 를 사용하면 보장해준다.
- Pushing out Data 의 경우 최소 한번은 일단 보장해준다. 데이터 전송은 여러번 할 수 있으니, 그치만 정확하게 한번만 보내기 위해서는 외부 시스템이 제공해주거나 제공해주지 않는다면 자체적인 트랜잭션 로직을 구현해야하거나 멱등성을 보장하도록 만들어야 할 수도 있다.

## **Semantics of Received Data**

Different input sources provide different guarantees, ranging from *at-least once* to *exactly once*. Read for more details.

### **With Files**

If all of the input data is already present in a fault-tolerant file system like HDFS, Spark Streaming can always recover from any failure and process all of the data. This gives *exactly-once* semantics, meaning all of the data will be processed exactly once no matter what fails.

### **With Receiver-based Sources**

For input sources based on receivers, the fault-tolerance semantics depend on both the failure scenario and the type of receiver. As we discussed [earlier](https://spark.apache.org/docs/latest/streaming-programming-guide.html#receiver-reliability), there are two types of receivers:

1. *Reliable Receiver* - These receivers acknowledge reliable sources only after ensuring that the received data has been replicated. If such a receiver fails, the source will not receive acknowledgment for the buffered (unreplicated) data. Therefore, if the receiver is restarted, the source will resend the data, and no data will be lost due to the failure.
2. *Unreliable Receiver* - Such receivers do *not* send acknowledgment and therefore *can* lose data when they fail due to worker or driver failures.

Depending on what type of receivers are used we achieve the following semantics. If a worker node fails, then there is no data loss with reliable receivers. With unreliable receivers, data received but not replicated can get lost. If the driver node fails, then besides these losses, all of the past data that was received and replicated in memory will be lost. This will affect the results of the stateful transformations.

To avoid this loss of past received data, Spark 1.2 introduced *write ahead logs* which save the received data to fault-tolerant storage. With the [write-ahead logs enabled](https://spark.apache.org/docs/latest/streaming-programming-guide.html#deploying-applications) and reliable receivers, there is zero data loss. In terms of semantics, it provides an at-least once guarantee.

The following table summarizes the semantics under failures:

[Summarize](https://www.notion.so/4845fc0237cf4fd0a585e90ee1be8b41)

### **With Kafka Direct API**

In Spark 1.3, we have introduced a new Kafka Direct API, which can ensure that all the Kafka data is received by Spark Streaming exactly once. Along with this, if you implement exactly-once output operation, you can achieve end-to-end exactly-once guarantees. This approach is further discussed in the [Kafka Integration Guide](https://spark.apache.org/docs/latest/streaming-kafka-0-10-integration.html).

- Input Source 를 File 로 이용하는 경우, 하둡과 같은 것들을 이용하는 경우에는 data loss 가 없고 한번만 딱 가지고 오는게 가능하다.
- 하지만 Input Source 를 Receiver 를 통해서 가지고 오는 경우에는 Receiver 의 종류를 고려해야한다. 데이터를 한번만 가지고 오는건 별로 문제가 되지 않을 것이다. (최대 한 번 처리)
- Receiver 는 Reliable Receiver, Unreliable Receiver 가 있는데 일반적으로 데이터를 읽어버릴 확률이 크 다.
- 이를 위해 spark 1.2 부터 들어온 write-ahead log 때문에 zero data loss 가 가능해졌다.
- 카프카를 Receiver 로 쓰는 경우에는 스파크 1.3 부터 안전하게 정확하게 한 번 가지고 오는게 가능하다.

## Semantics of output operations

Output operations (like `foreachRDD`) have *at-least once* semantics, that is, the transformed data may get written to an external entity more than once in the event of a worker failure. While this is acceptable for saving to file systems using the `saveAs***Files` operations (as the file will simply get overwritten with the same data), additional effort may be necessary to achieve exactly-once semantics. There are two approaches.

- *Idempotent updates*: Multiple attempts always write the same data. For example, `saveAs***Files` always writes the same data to the generated files.
- *Transactional updates*: All updates are made transactionally so that updates are made exactly once atomically. One way to do this would be the following.
    - Use the batch time (available in `foreachRDD`) and the partition index of the RDD to create an identifier. This identifier uniquely identifies a blob data in the streaming application.
    - Update external system with this blob transactionally (that is, exactly once, atomically) using the identifier. That is, if the identifier is not already committed, commit the partition data and the identifier atomically. Else, if this was already committed, skip the update.

```scala
dstream.foreachRDD { (rdd, time) =>
  rdd.foreachPartition { partitionIterator =>
    val partitionId = TaskContext.get.partitionId()
    val uniqueId = generateUniqueId(time.milliseconds, partitionId)
    // use this uniqueId to transactionally commit the data in partitionIterator
  }
}
```

- Pushing out Data 의 경우는 최소 한 번 보장일 것이다. 다시 시작하는 경우가 있을수도 있고, 동시성 문제가 생길수도 있으니.
- 일반적인 파일로 저장하는 `saveAsXXXFiles` 의 오퍼레이션은 파일을 overwrtting 한다. 그러므로 동시성 문제가 생겼을 때 문제가 될 수 있으니 멱등성을 보장하도록 설계하거나 트랜잭션을 보장하도록 해야한다.
    - spark streaming 에서 트랜잭션을 보장하려면 RDD Partition index 를 이용해서 유니크한 ID 를 만어서 이용하면 된다. 현재 데이터가 커밋되어있지 않다면 커밋하고, 커밋되어 있으면 스킵하고 그런 식으로 결국 최종 RDD 는 변하지 않을 것이니 이를 이용하면 된다.

