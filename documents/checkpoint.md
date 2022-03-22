# Spark Checkpoint

[https://spark.apache.org/docs/latest/streaming-programming-guide.html#checkpointing](https://spark.apache.org/docs/latest/streaming-programming-guide.html#checkpointing)

## ****Checkpointing****

A streaming application must operate 24/7 and hence must be resilient to failures unrelated to the application logic (e.g., system failures, JVM crashes, etc.). For this to be possible, Spark Streaming needs to *checkpoint* enough information to a fault- tolerant storage system such that it can recover from failures. There are two types of data that are checkpointed.

- *Metadata checkpointing:*
    - Saving of the information defining the streaming computation to fault-tolerant storage like HDFS. This is used to recover from failure of the node running the driver of the streaming application (discussed in detail later). Metadata includes:
    - *Configuration* - The configuration that was used to create the streaming application.
    - *DStream operations* - The set of DStream operations that define the streaming application.
    - *Incomplete batches* - Batches whose jobs are queued but have not completed yet.
- *Data checkpointing* - Saving of the generated RDDs to reliable storage. This is necessary in some *stateful* transformations that combine data across multiple batches. In such transformations, the generated RDDs depend on RDDs of previous batches, which causes the length of the dependency chain to keep increasing with time. To avoid such unbounded increases in recovery time (proportional to dependency chain), intermediate RDDs of stateful transformations are periodically *checkpointed* to reliable storage (e.g. HDFS) to cut off the dependency chains.

To summarize, metadata checkpointing is primarily needed for recovery from driver failures, whereas data or RDD checkpointing is necessary even for basic functioning if stateful transformations are used.

- 스파크 어플리케이션이 실패했을 때 탄력적으로 복구하기 위해서 Checkpoint 가 존재한다.
- 체크포인트에서 저장되는 데이터 유형을 크게 두 가지가 있다.
    - Metadata checkpointing: Streaming 계산을 정의하는 정보를 HDFS 같은 fault-tolerant 스토로지에 저장한다. (fault-tolerant 는 결함이 나도 시스템이 올바르게 작동할 수 있다는 의미다.)
    - Metadata 는 Configuration 과 DStream operations, Incomplete batches 정보를 포함한다.
        - Configuration: Stream 을 만드는데 사용하는 설정 정보.
        - DStream operation: Stream 어플리케이션에서 사용하는 DStream operation 의 집합들.
        - Incomplete batches: 큐에 있지만 아직 완료되지 않은 배치들.
    - Data checkpointing: 만들어진 RDD 를 Reliable storage 에 저장한다. 이건 Stateful transformation 에 필요하다. (이전 RDD 에 의존하는 그런 유형의 데이터를 말하는 듯.)
- 정리하자면 metadata 는 driver 의 실패로부터 복구할 때 필요하고 RDD or data 는 Stateful transformation 에 필요하다.