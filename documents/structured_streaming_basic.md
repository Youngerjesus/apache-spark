# Structured Streaming Basic

### 구조적 스트리밍 기초

- 구조적 스트리밍 API 의 핵심 아이디어는 추가적으로 들어오는 스트림 데이터를 테이블 처럼 다루는 것이다.
    - 들어오는 데이터는 테이블에 레코드가 추가되는 것과 같다.
    - 필요한 경우 상태 저장소에 있는 일부 상태를 갱신해서 결과를 변경한다.
    - 이렇게 함으로써 배치 처리와 스트림 처리의 코드는 같게 된다. 따로 코드 관리를 하지 않아도 된다.


## 구조적 스트리밍 핵심 개념

- 구조적 스트리밍 잡과 관련된 몇가지 핵심 개념을 알아보자.

### 트랜스포메이션과 액션

- 트랜스포메이션은 배치처리에서 했던 것과 동일하다. 하지만 몇가지 제약사항이 있는데 증분 처리를 할 수 없는 일부 쿼리 유형이 있다.
- 구조적 스트리밍에서는 스트림 처리를 한 뒤 결과를 출력하는 한 가지 액션만 있다.

### 입력 소스

- 구조적 스트리밍은 스트리밍 방식으로 데이터를 읽을 수 있는 몇 가지 입력 소스를 지원한다.
    - 아파치 카프카
    - HDFS 나 S3
    - 테스트용 소켓 소스

### 싱크

- 입력 소스로 데이터를 얻듯이 싱크로 스트림의 결과를 저장할 목적지를 명시한다.
- 스파크 2.2 버전에서 지원하는 출력용 싱크는 다음과 같다.
    - 아파치 카프카
    - 거의 모든 파일 포맷
    - 출력 레코드에 임의 연산을 실행하는 foreach 싱크
    - 테스트용 콘솔 싱크
    - 디버깅용 메모리 싱크

### 출력 모드

- 구조적 스트리밍에서는 싱크를 정의하기 위해서 데이터를 출력하는 형태를 정의하는 출력 모드를 정의해야한다.
- 다음과 같은 출력 모드가 있다.
    - append: 싱크에 신규 정보만 추가하는 경우
    - update: 바뀐 정보로 기본 레코드를 갱신
    - complete: 매번 전체 결과를 재작성
- 여기서 알아야 할 건 특정 쿼리와 싱크는 일부 출력 모드만 지원한다는 점이다.
- 예로 스트림에 map 연산을 수행하는 잡이 있다고 생각해보면 complete 로 실행할 경우 문제가 생긴다. 출력 데이터가 매번 새롭게 큰 것들이 만들어 질 것이니.
- 반면 한정된 수의 키를 사용해 집계한 후 시간에 따라 갱신해야 한다면 append 보다는 update 나 complete 가 적합하다.

### 트리거

- 출력 모드가 데이터 출력 방식을 정의한다면 트리거는 데이터 출력 시점을 정의한다.
- 트리거는 구조적 스트리밍에서 언데 결과를 갱신할지 정의하는데 기본적으로 마지막 입력 데이터를 처리한 직후 신규 입력 데이터를 조회해 최단 시간내에 새로운 처리 결과를 만들어낸다.
- 하지만 이런 동작 방식 떄문에 파일 싱크를 이용하다면 작은 크기의 파일이 여러 개 생길 수 있다.
- 따라서 스파크는 처리 시간 (고정된 주기로만 신규 데이터를 탐색하는) 트리거도 지원한다.
- 처리 시간 기반 트리거

```scala
activityCounts.writeStream.trigger(Trigger.ProcessingTime("100 seconds"))
	.format("console")
	.outputMode("complete")
	.start()
```

- 일회성 트리거

```scala
activityCounts.writeStream.trigger(Trigger.once())
	.format("console")
	.outputMode("complete")
	.start()
```

- 운영환경에서 수동으로 실행할 때 사용하거나,
- 개발중에는 한번에 처리할 수 있는 수준의 데이터의 경우에 사용한다.

### 이벤트 시간 처리

- 구조적 스트리밍은 이벤트 시간 기준의 처리도 지원한다.
- 이 처리 방식은 무작위로 도착한 레코드 내부에 기록된 타임 스탬프를 기준으로 처리한다.
- 여기서 알아야 할 개념으로 두 가지가 있는데 이건 이후에도 자세히 설명하겠다.
    - 이벤트 시간 데이터
        - 이벤트 시간을 데이터의 타임 스탬프 필드로 생각한다고 했었다. 스파크 스트리밍에서 데이터는 테이블에 추가로 들어온 레코드와 같으므로 타임 스탬프 컬럼을 기준으로 집계 및 그룹화 해서 윈도우 처리를 하면된다. 이런 작업을 제어할 땐 워터마크를 사용한다.
    - 워터마크
        - 워터마크는 시간 제한을 설정할 수 있는 스트리밍 시스템의 기능이다. 늦게 들어온 이벤트를 어디까지 처리할 지 시간을 제한할 수있다.
        - 한 가지 예로 모바일 장비의 로그를 처리하는 애플리케이션에서 업로드 지연 현상 때문에 30분 전 데이터까지 처리해야 하는 경우도 있다.
        - 워터마크는 특정 이벤트 시간의 윈도우 결과를 출력하는 시점을 제어할 때도 사용한다.


## 구조적 스트리밍 활용

- 구조적 스트리밍을 실전에서 어떻게 활용할 수 있는지 예제로 보자.
- 이 예제는 이기종 데이터셋을 사용한다.
- 데이터는 스마트폰과 스마트워치에서 지원하는 최대 빈도로 샘플링한 센서 데이터로 구성되어 있다.
- 센서 데이터는 사용자가 자전거 타기, 앉기, 일어서기, 걷기 등의 활동을 하는동안 기록한 것을 말한다.

### 예제

스트리밍으로 데이터를 읽는 코드는 다음과 같다.

```scala
val streaming = spark.readStream.schema(dataSchema)
									.option("maxFilesPerTrigger",1)
									.json("/data/activity-data")
```

- DataFrame 에서 스키마 추론 기능을 사용할려면 spark.sql.streaming.schemaInference 설정을 true 로 설정하면 된다. 하지만 모르는 사이에 데이터는 변경될 가능성이 있으니 운영환경에서는 스키마 추론 방식을 사용하면 안된다.
- maxFilesPerTrigger 는 폴더 내의 파일을 얼마나 빠르게 읽을지 결정하는 값이다. 이 값을 낮게 잡으면 스트림의 흐름을 인위적으로 제한하는게 가능하다. 여기서는 1 로 잡아서 증분 처리를 쉽게 보여주도록 했다.

다음은 groupBy 트랜스포메이션을 사용한 예제다.

- gt 칼럼을 기준으로 그룹화하고 데이터 수를 계산한다.

```scala
val activityCounts = streaming.groupBy("gt").count() 
```

- 스트림의 데이터 포메이션은 다른 스파크 API 처럼 지연 처리 방식으로 동작한다.

트랜스 포메이션을 정의했으니 액션을 정의해보자. 여기서는 결과를 내보낼 목적지로 메모리를 선택했다.

그리고 싱크를 정의했으니 출력 모드도 정의해야하는데 여기서는 모든 키와 데이터 수를 다시 저장하는 complete 출력 모드를 사용했다.

```scala
val activityQuery = activityCounts.writeStream.queryName("activity_counts")
	.format("memory")
	.outputMode("complete")
	.start()
```

코드 예제를 실행할려면 다음 코드도 추가해야한다.

```scala
activityQuery.awaitTermination()
```

- 코드를 실행하면 백그라운드에서 스트리밍 연산이 실행하고 종료까지 대기할 수 있도록 awaitTermination() 을 지정했다.

실행중인 스트림 목록은 SparkSession 에서 확인하는게 가능하다.

```scala
spark.streams.active
```

- 스파크는 각 스트림에 UUID 를 붙인다.
- 필요한 경우 스트림 목록을 조회하는게 가능하다. 여기서는 변수에 할당했으므로 그럴 필요가 없긴하다.

스트리밍 집계 결과가 저장된 메모리 테이블을 조회해보자.

```scala
for (i <- 1 to 5) {
	spark.sql("SELECT * FROM activity_counts").show()
	Thread.sleep(1000)
}
```

- 쿼리를 1 초마다 출력하는 반복문을 사용했다.

## 스트림 트랜스포메이션

### 선택과 필터링

- 구조적 스트리밍은 DataFrame 의 모든 함수와 개별 컬럼을 처리하는 선택과 필터링 그리고 단순한 트랜스포메이션을 지원한다.

```scala
val simpleTransform = streaming.withColumn("stairs", expr("gt like '%stairs%'"))
	.where("stairs")
	.where("gt is not null")
	.select("gt", "model", "arrival_time", "cration_time")
	.writeStream
	.queryName("simple_transform")
	.format("memory")
	.outputMode("append")
	.start()
```

### 카프카 소스에서 메시지 읽기

- 메시지를 읽기 위해서는 다음 옵션 중 하나를 선택해야한다.
    - assign
    - subscribe
    - subscribePattern
- 카프카에서 메시지를 읽을 때는 이 옵션 중  하나만 지정해야 한다.
    - asssign 은 토픽 뿐 아니라 파티션까지 세밀하게 지정하는 옵션이다.
    - subscribe 와 subscribePattern 은 각각 토픽 목록과 패턴을 지정해서 여러 토픽을 구독하는 옵션이다.
- 두 번째로 할 일은 카프카에 접속할 수 있도록 kafka.bootstrap.servers 값을 지정하는 것이다.
- 이것 외에도 설정한 옵션은 다음과 같다.
    - startingOffsets 및 endingOffsets
        - 쿼리를 시작할 때 읽을 지점을 말한다.
        - 가장 작은 오프셋부터 읽는 earliest 와
        - 가장 큰 포으펫부터 읽는 latest 가 있다.
        - JSON 에 오프셋을 -2 로 지정하면 earliest 로 -1 로 지정하면 latest 로 동작한다.
    - failOnDataLoss
        - 데이터 유실이 일어났을 때 (토픽이 삭제되거나 오프셋이 범위를 벗어났을 때) 쿼리를 중단할 것인지 지정하는 것이다.
        - 잘못된 정보를 줄 수 있으므로 원하는 대로 동작하지 않는 경우 비활성화 하는게 가능하다. 기본값은 true 이다.
    - maxOffsetsPerTrigger
        - 특정 트리거 시점에 읽을 오프셋 개수다.
- 예제 코드는 다음과 같다.

```scala
// topic 1 수신
val ds1 = spark.readStream.,format("kafka")
	.option("kafka.bootstrap.servers", "host1:port1, host2:port2")
	.option("subscribe", "topic1")
	.load()

// topic 1, topic 2 수신
val ds2 = spark.readStream.,format("kafka")
	.option("kafka.bootstrap.servers", "host1:port1, host2:port2")
	.option("subscribe", "topic1,topic2")
	.load()

// 특정 패턴에 맞는 토픽 수신
val ds3 = spark.readStream.,format("kafka")
	.option("kafka.bootstrap.servers", "host1:port1, host2:port2")
	.option("subscribePattern", "topic.*")
	.load()
```

- 카프카 소스의 각 로우는 다음과 같은 스키마를 가진다.
    - key: binary
    - value: binary
    - topic: string
    - pattern: int
    - offset: long
    - timestamp: long

### 테스트용 소스와 싱크

- 스파크는 스트리밍 쿼리의 프로토 타입을 만들거나 디버깅 시 유용한 테스트 소스와 싱크를 제공한다.
- 이건 개발할 때만 사용하면 된다.
- 소켓 소스

```scala
val socketDF = spark.readStream.format("socket")
	.option("host", "localhost")
	.option("port", 9999)
	.load()
```

- TCP 소켓을 통해서 스트림 데이터를 전송하는게 가능하다.
- 이걸 사용할려면 데이터를 읽기 위해 호스트와 포트를 지정해줘야한다.
- 메모리 싱크

```scala
activityCounts.writeStream.format("memory")
	.queryName("my_device_table")
```

- 이 싱크도 개발용에서만 사용해야 한다. 내고장성을 제공하지 않으므로.