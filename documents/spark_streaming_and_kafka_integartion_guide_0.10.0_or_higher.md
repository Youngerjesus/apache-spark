# Spark Streaming + Kafka Integartion Guide (Kafka broker version 0.10.0 or higher)

[https://spark.apache.org/docs/latest/streaming-kafka-0-10-integration.html](https://spark.apache.org/docs/latest/streaming-kafka-0-10-integration.html)

The Spark Streaming integration for Kafka 0.10 provides simple parallelism, 1:1 correspondence between Kafka partitions and Spark partitions, and access to offsets and metadata. However, because the newer integration uses the [new Kafka consumer API](https://kafka.apache.org/documentation.html#newconsumerapi) instead of the simple API, there are notable differences in usage.

- Kafka 0.10 는 Kafka partition 과 Spark partition 과의 1:1 매칭을 지원해준다.
- 그리고 새로운 통갑 가이드는 Spark Consumer API 를 사용하는게 가능하다. 기존에는 Simple API 를 통해서 사용이 가능했음. (Simple API 는 그냥 topic + parittion 으로 주기적으로 데이터를 가지고 오는 것. 주키퍼에서 offset 관리를 하지 않음.). 정의를 보니 컨슈머를 사용한것도 아닌듯.

## ****Linking****

For Scala/Java applications using SBT/Maven project definitions, link your streaming application with the following artifact (see [Linking section](https://spark.apache.org/docs/latest/streaming-programming-guide.html#linking) in the main programming guide for further information).

```xml
groupId = org.apache.spark
artifactId = spark-streaming-kafka-0-10_2.12
version = 3.2.1
```

**Do not** manually add dependencies on `org.apache.kafka` artifacts (e.g. `kafka-clients`). The `spark-streaming-kafka-0-10` artifact has the appropriate transitive dependencies already, and different versions may be incompatible in hard to diagnose ways.

- org.apache.kafka 를 수동으로 추가하지마라. spark-streaming-kafka-0-10 에 다 들어가있다.

## ****Creating a Direct Stream****

Note that the namespace for the import includes the version, org.apache.spark.streaming.kafka010.

```scala
import org.apache.kafka.clients.consumer.ConsumerRecord
import org.apache.kafka.common.serialization.StringDeserializer
import org.apache.spark.streaming.kafka010._
import org.apache.spark.streaming.kafka010.LocationStrategies.PreferConsistent
import org.apache.spark.streaming.kafka010.ConsumerStrategies.Subscribe

val kafkaParams = Map[String, Object](
  "bootstrap.servers" -> "localhost:9092,anotherhost:9092",
  "key.deserializer" -> classOf[StringDeserializer],
  "value.deserializer" -> classOf[StringDeserializer],
  "group.id" -> "use_a_separate_group_id_for_each_stream",
  "auto.offset.reset" -> "latest",
  "enable.auto.commit" -> (false: java.lang.Boolean)
)

val topics = Array("topicA", "topicB")
val stream = KafkaUtils.createDirectStream[String, String](
  streamingContext,
  PreferConsistent,
  Subscribe[String, String](topics, kafkaParams)
)

stream.map(record => (record.key, record.value))
```

Each item in the stream is a [ConsumerRecord](http://kafka.apache.org/0100/javadoc/org/apache/kafka/clients/consumer/ConsumerRecord.html)

For possible kafkaParams, see [Kafka consumer config docs](http://kafka.apache.org/documentation.html#newconsumerconfigs). If your Spark batch duration is larger than the default Kafka heartbeat session timeout (30 seconds), increase heartbeat.interval.ms and session.timeout.ms appropriately. For batches larger than 5 minutes, this will require changing group.max.session.timeout.ms on the broker. Note that the example sets enable.auto.commit to false, for discussion see [Storing Offsets](https://spark.apache.org/docs/latest/streaming-kafka-0-10-integration.html#storing-offsets) below.

- 네임스페이스를 주목하자. org.apache.spark.streaming.kafka010
- 고려해야하는 옵션이 ``heartbeat.interval.ms`` 와 ``session.timeout.ms`` 그리고 ``max.poll.interval.ms`` 마지막으로 ``group.max.session.timeout.ms`` 이다.
    - ``heartbeat.interval.ms`` 는 컨슈머 코디네이터에게 하트비트를 보내는 시간이다. 이 값은 ``session.timeout.ms`` 보다 작아야 하며 1/3 이 적당하다. 기본적으로 Spark 에서는 5 분으로 세팅되는 듯.
    - ``group.max.session.timeout.ms`` 는 session timeout 에 설정할 수 있는 최대값을 말한다.
    - ``max.poll.interval.ms`` 이 시간안에 poll() 을 호출하지 않으면 장애가 있다고 판단해서 리밸런싱 진행함. 기본 값은 5 분.
- spark batch 동안에 hearbeat 를 보내지 못하는 거 같다.

## ****LocationStrategies****

The new Kafka consumer API will pre-fetch messages into buffers. Therefore it is important for performance reasons that the Spark integration keep cached consumers on executors (rather than recreating them for each batch), and prefer to schedule partitions on the host locations that have the appropriate consumers.

In most cases, you should use `LocationStrategies.PreferConsistent` as shown above. This will distribute partitions evenly across available executors. If your executors are on the same hosts as your Kafka brokers, use `PreferBrokers`, which will prefer to schedule partitions on the Kafka leader for that partition. Finally, if you have a significant skew in load among partitions, use `PreferFixed`. This allows you to specify an explicit mapping of partitions to hosts (any unspecified partitions will use a consistent location).

The cache for consumers has a default maximum size of 64. If you expect to be handling more than (64 * number of executors) Kafka partitions, you can change this setting via `spark.streaming.kafka.consumer.cache.maxCapacity`

The cache is keyed by topicpartition and group.id, so use a **separate** `group.id` for each call to `createDirectStream`.

- Spark Consumer API 에서는 메시지를 미리 버퍼로 가지고 온다. 그리고 매번 배치 때마다 컨슈머를 만드는게 아니라 Consumer 를 Executor 캐시에 보관해두는게 성능 상으로 더 낫다.
- Consumer 가 있는 Executor 에서 파티션을 가지고오고 그걸 Executor 에게 분산하는 식으로 처리하는 방식인듯?
- 대부분의 경우에는 ``LocationStrategies.PreferConsistent`` 를 사용하면 된다. 이건 Consumer 에서 가지고 온 데이터를 RDD 파티션으로 처리해서 처리 가능한 모든 Executor 에게 분배하는 식으로 처리한다.
- 만약 Executor 가 Kafka Broker 와 같은 노드에 있다면 ``LocationStrategies.PreferBrokers`` 를 하면 된다. 그러면 카프카의 리더 파티션으로부터 파티션을 만드는 듯.
- Kafka 파티션의 부하 차이가 많이 난다면 ``LocationStrategies.PreferFixex`` 를 통해서 각 파티션을 특정한 호스트에 명시적으로 매핑하는 것도 가능함.
- 컨슈머를 캐시로 가지고 있을 수 있는 최대 개수는 64 개임. 수정하고 싶다면 ``spark.streaming.kafka.consumer.cache.maxCapacity`` 를 통해서 수정하면 됨.
- 캐시는 topicpatition 과 [group.id](http://group.id) 를 기반으로 지정되니까 createDirectStream 을 만들 땐 group id 별로 구별하는게 좋음.

## ****ConsumerStrategies****

The new Kafka consumer API has a number of different ways to specify topics, some of which require considerable post-object-instantiation setup. `ConsumerStrategies` provides an abstraction that allows Spark to obtain properly configured consumers even after restart from checkpoint.

`ConsumerStrategies.Subscribe`, as shown above, allows you to subscribe to a fixed collection of topics. `SubscribePattern` allows you to use a regex to specify topics of interest. Note that unlike the 0.8 integration, using `Subscribe` or `SubscribePattern` should respond to adding partitions during a running stream. Finally, `Assign` allows you to specify a fixed collection of partitions. All three strategies have overloaded constructors that allow you to specify the starting offset for a particular partition.

If you have specific consumer setup needs that are not met by the options above, `ConsumerStrategy` is a public class that you can extend.

- Kafka Consumer API 는 토픽을 지정하는 방식이 여러가지가 있는데 그 들 중 대부분이 객체 초기화 이후에 설정된다.
- ``ConsumerStrategies`` 는 Spark 가 checkpoint 로 부터 restart 될 때 적절한 Consumer 를 얻도록 하는 추상화 방식을 제공해준다. (여기서 “적절한" 의 의미는 토픽이 지정된 컨슈머를 말하는 듯)
- ``ConsumerStrategies.Subscribe`` 는 고정된 컬렉션을 바탕으로 토픽을 지정할 수 있는 반면에 ``SubscribePattern`` 은 정규표현식으로 토픽을 지정할 수 있다.
- ``Assign`` 의 경우에는 고정된 파티션을 지정하는 것도 가능하다. 그리고 세 전략 모두다 특정한 파티션의 시작하는 오프셋을 지정하는게 가능하다.
- 마냥ㄱ에 원하는 Consumer Setup 이 없다면 새롭게 커스텀으로 확잔하는 것도 가능하다.

## ****Creating an RDD****

If you have a use case that is better suited to batch processing, you can create an RDD for a defined range of offsets.

```scala
// Import dependencies and create kafka params as in Create Direct Stream above

val offsetRanges = Array(
  // topic, partition, inclusive starting offset, exclusive ending offset
  OffsetRange("test", 0, 0, 100),
  OffsetRange("test", 1, 0, 100)
)

val rdd = KafkaUtils.createRDD[String, String](sparkContext, kafkaParams, offsetRanges, PreferConsistent)
```

Note that you cannot use `PreferBrokers`, because without the stream there is not a driver-side consumer to automatically look up broker metadata for you. Use `PreferFixed` with your own metadata lookups if necessary.

## ****Obtaining Offsets****

```scala
stream.foreachRDD { rdd =>
  val offsetRanges = rdd.asInstanceOf[HasOffsetRanges].offsetRanges
  rdd.foreachPartition { iter =>
    val o: OffsetRange = offsetRanges(TaskContext.get.partitionId)
    println(s"${o.topic} ${o.partition} ${o.fromOffset} ${o.untilOffset}")
  }
}
```

Note that the typecast to `HasOffsetRanges` will only succeed if it is done in the first method called on the result of `createDirectStream`, not later down a chain of methods. Be aware that the one-to-one mapping between RDD partition and Kafka partition does not remain after any methods that shuffle or repartition, e.g. reduceByKey() or window().

- ``HasOffsetRanges`` 는 ``createDirectStream`` 을 만든 후 처음 실행할 때 rdd 로 부터 typecast 를 해서 얻을 수 있다. (이 HasOffsetRanges 로부터 offset 을 얻을 수 있음.) 추가적인 작업을 하면 얻을 수 없음. 딱 처음 가져올 때 Kafka Partition 과 RDD Partition 이 1:1 매핑되기 때문에 Shuffle 을 하거나 repartition 을 하면 얻을 수 없다.

## ****Storing Offsets****

Kafka delivery semantics in the case of failure depend on how and when offsets are stored. Spark output operations are [at-least-once](https://spark.apache.org/docs/latest/streaming-programming-guide.html#semantics-of-output-operations). So if you want the equivalent of exactly-once semantics, you must either store offsets after an idempotent output, or store offsets in an atomic transaction alongside output. With this integration, you have 3 options, in order of increasing reliability (and code complexity), for how to store offsets.

### ****Checkpoints****

If you enable Spark [checkpointing](https://spark.apache.org/docs/latest/streaming-programming-guide.html#checkpointing), offsets will be stored in the checkpoint. This is easy to enable, but there are drawbacks. Your output operation must be idempotent, since you will get repeated outputs; transactions are not an option. Furthermore, you cannot recover from a checkpoint if your application code has changed. For planned upgrades, you can mitigate this by running the new code at the same time as the old code (since outputs need to be idempotent anyway, they should not clash). But for unplanned failures that require code changes, you will lose data unless you have another way to identify known good starting offsets.

### ****Kafka itself****

Kafka has an offset commit API that stores offsets in a special Kafka topic. By default, the new consumer will periodically auto-commit offsets. This is almost certainly not what you want, because messages successfully polled by the consumer may not yet have resulted in a Spark output operation, resulting in undefined semantics. This is why the stream example above sets “enable.auto.commit” to false. However, you can commit offsets to Kafka after you know your output has been stored, using the `commitAsync` API. The benefit as compared to checkpoints is that Kafka is a durable store regardless of changes to your application code. However, Kafka is not transactional, so your outputs must still be idempotent.

```scala
stream.foreachRDD { rdd =>
  val offsetRanges = rdd.asInstanceOf[HasOffsetRanges].offsetRanges

  // some time later, after outputs have completed
  stream.asInstanceOf[CanCommitOffsets].commitAsync(offsetRanges)
}
```

As with HasOffsetRanges, the cast to CanCommitOffsets will only succeed if called on the result of createDirectStream, not after transformations. The commitAsync call is threadsafe, but must occur after outputs if you want meaningful semantics.

### ****Your own data store****

For data stores that support transactions, saving offsets in the same transaction as the results can keep the two in sync, even in failure situations. If you’re careful about detecting repeated or skipped offset ranges, rolling back the transaction prevents duplicated or lost messages from affecting results. This gives the equivalent of exactly-once semantics. It is also possible to use this tactic even for outputs that result from aggregations, which are typically hard to make idempotent.

```scala
// The details depend on your data store, but the general idea looks like this

// begin from the offsets committed to the database
val fromOffsets = selectOffsetsFromYourDatabase.map { resultSet =>
  new TopicPartition(resultSet.string("topic"), resultSet.int("partition")) -> resultSet.long("offset")
}.toMap

val stream = KafkaUtils.createDirectStream[String, String](
  streamingContext,
  PreferConsistent,
  Assign[String, String](fromOffsets.keys.toList, kafkaParams, fromOffsets)
)

stream.foreachRDD { rdd =>
  val offsetRanges = rdd.asInstanceOf[HasOffsetRanges].offsetRanges

  val results = yourCalculation(rdd)

  // begin your transaction

  // update results
  // update offsets where the end of existing offsets matches the beginning of this batch of offsets
  // assert that offsets were updated correctly

  // end your transaction
}
```

- Kafka 가 실패로부터 작동할 땐 offset 이 어디까지 저장되었는지가 중요하다. Spark 은 최소 한 번 작동은 보장한다. (checkpoint 가 있어서.)
- 만약 정확하게 한 번을 처리하고 싶다면 offset 을 트랜잭션을 이용해서 저장을 하고 output opeartion 은 멱등성을 보장하도록 하자.
- checkpoint 를 사용할 때 주의할 점은 어플리케이션 코드가 변경되는 경우라면 사용할 수 없다라는 것.
- 카프카에 처리한 offset 을 기록할 수 있다. ``commitAsync`` 와 같은 API 를 통해서. 하지만 이것은 트랜잭션을 보장해주지 않기 때문에 멱등성 아웃풋을 내도록 해줘야한다.
- `HasOffsetRanges` 는 ``CanCommitOffsets`` 로 변경이 가능하다. 물론 transformation 이 되지 않았더라면

## ****SSL / TLS****

The new Kafka consumer [supports SSL](http://kafka.apache.org/documentation.html#security_ssl). To enable it, set kafkaParams appropriately before passing to `createDirectStream` / `createRDD`. Note that this only applies to communication between Spark and Kafka brokers; you are still responsible for separately [securing](https://spark.apache.org/docs/latest/security.html) Spark inter-node communication.

```scala
val kafkaParams = Map[String, Object](
  // the usual params, make sure to change the port in bootstrap.servers if 9092 is not TLS
  "security.protocol" -> "SSL",
  "ssl.truststore.location" -> "/some-directory/kafka.client.truststore.jks",
  "ssl.truststore.password" -> "test1234",
  "ssl.keystore.location" -> "/some-directory/kafka.client.keystore.jks",
  "ssl.keystore.password" -> "test1234",
  "ssl.key.password" -> "test1234"
)
```

- Kafka Consumer 에서는 SSL 을 지원한다. 이것을 활성화하면 Spark 와 Kafka Broker 끼리 통신할 때 안전하게 통신한다.

## ****Deploying****

As with any Spark applications, `spark-submit` is used to launch your application.

For Scala and Java applications, if you are using SBT or Maven for project management, then package `spark-streaming-kafka-0-10_2.12` and its dependencies into the application JAR. Make sure `spark-core_2.12` and `spark-streaming_2.12` are marked as `provided` dependencies as those are already present in a Spark installation. Then use `spark-submit` to launch your application (see [Deploying section](https://spark.apache.org/docs/latest/streaming-programming-guide.html#deploying-applications) in the main programming guide).

- `spark-submit`  으로 어플리케이션을 시장한다.
- SBT 나 Maven 을 이용한다면 ``spark-streaming-kafka-0-10_2.12`` 와 의존성들을 패키지해서 JAR 파일로 만들면 된다. 그 다음 ``spark-submit`` 하면 실행됨.  만약 Spark 가 설치되어 있는 곳이라면 `spark-core_2.12`  와 ``spark-streaming_2.12`` 를 provided 의존성을 주도록 하자.

## Security

See [Structured Streaming Security](https://spark.apache.org/docs/latest/structured-streaming-kafka-integration.html#security).