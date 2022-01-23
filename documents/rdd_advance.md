# RDD 고급

여기서는 키-값 형태의 RDD 를 중심으로 RDD 고급 연산을 보겠다.

또산 사용자 정의 파티션과 같은 개념도 알아보겠다. 사용자 정의 파티션을 이용하면 클러스터에 데이터가 배치되는 방식을 정확히 제어할 수 있고 개별 파티션을 다룰 수 있다.

## 키-값 형태의 RDD

RDD 에는 데이터를 키-값 형태로 다룰 수 있는 다양한 메소드가 있다. 이러한 메소드의 이름은 <연산명>ByKey 형태의 이름을 가진다.

메소드 이름에 ByKey 가 있다면 PairRdd 타입만 사용이 가능하다.

PairRDD 타입을 만드는 가장 쉬운 방법은 RDD map 연산에 키-값 구조로 만드는 것이다.

즉 RDD 레코드에 두 개의 값이 존재한다. 예제로보자.

```scala
words.map(word => (word.toLowerCase, 1)
```

### keyBy

이전 예제에서 PairRDD 를 만드는 간단한 방법을 알아봤다.

다른 방법으로는 현재 값으로부터 키를 생성하는 keyBy 함수를 통해 동일한 결과를 얻을 수 있다.

다음 예제는 단어의 첫 번째 문자를 키로 만들어서 RDD 를 생성하는 코드다.

```scala
val keyword = words.keyBy(word => word.toLowerCase.toSeq(0).toString)
```

### 값 매핑하기

위의 예제에서 키-값 데이터를 다뤄보자.

만약 튜플 형태의 데이터를 사용한다면 스파크는 첫 번째 요소를 키로, 두 번째 요소를 값으로 추정한다.

튜플 형태의 데이터에서 키를 제외하고 값만 추출하는 것도 가능하다.

다음 예제와 같이 mapValues 메소드를 통해서 값 수정 시 발생할 수 있는 오류를 미리 방지해보자.

```scala
keyword.mapValues(word => word.toUpperCase).collect()
```

### 키와 값 추출하기

키-값 형태의 데이터를 가지고 있다면 다음 메소드를 통해 키나 값 전체를 추출할 수 있다.

```scala
keyword.keys.collect()
keyword.values.collect()
```

### lookup

RDD 를 사용해서 할 수 있는 작업중에는 특정 키에 관한 결과를 찾는 것이다.

그러나 각 입력에 대해서 오로지 하나의 키만 찾을 수 있도록 하는 방법은 없다.

loopup 함수를 통해서 인수로 ‘s’ 를 입력하면 키가 ‘s’ 인 결과들을 찾는데 ‘spark’ 나 ‘simple’ 등이 반환될 것이다.

```scala
keyword.lookup('s')
```

## 집계

사용하는 메소드에 따라 일반 RDD 나 PairRDD 를 사용해서 집계를 수행할 수 있다.

여기서의 예제들은 아래의 데이터셋을 이용해서 설명하겠다.

```scala
val chars = words.flatMap(word => word.toLowerCase.toSeq)
val KVcharacters = chars.map(letter => (letter,1))

def maxFunc(left:Int, right:Int) = math.max(left,right)
def addFunc(left:Int, right:Int) = left + right
```

### countByKey

countByKey 메소드는 각 키의 아이템 수를 구하고 로컬 맵으로 결과를 수집한다.

스칼라나 자바를 사용한다면 제한 시간과 신뢰도를 인수로 지정해서 근사치를 구할 수도 있다.

### 집계 연산 구현 방식 이해하기

PairRDD 를 생성하는 몇 가지 방법이 있는데 이때 구현 방식이 다를 수 있다.

이런 구현 방식은 Job 의 안정성을 위해 중요한데

이 차이를 groupBy 와 reduce 함수를 통해서 비교해보자.

groupBy 와 reduce 모두 동일한 기본원칙이 적용되므로 키를 기준으로 비교해보자.

### groupByKey

각 키의 레코드 총 수를 구하라고 했을 때 groupByKey 를 통해 각 그룹에 map 연산을 통해 처리할 수 있다.

```scala
KVcharacters.groupByKey().map(row => (row._1, row._2.reduce(addFunc))).collect()
```

이 경우의 문제점은 모든 Executor 에서 함수를 적용하기 전에 해당 키와 관련된 모든 값을 메모리로 읽어 들인다는 점이다.

그러므로 크기가 큰 데이터의 경우 OutofMememoryError 가 발생할 수 있다.

### reduceByKey

reduceByKey 를 통해서 결과값을 모을 수 있는데 이렇게 할 경우 각 파티션에서 reduce 작업을 할 수 있기 떄문에 모든 값을 메모리에 유지하지 않아도 된다.

또한 최종 리듀스 과정을 제외한 모든 작업은 개별 워커에서 처리하기 때문에 연산 중 셔플이 발생하지도 않는다.

파티션은 스파크에서 병렬로 처리할 수 있는 최소 데이터 모음 단위이다.

셔플은 스파크에서 re-partitioning 을 하는 행위를 말한다.

그러므로 이 방법을 사용하면 안정성과 연산 속도가 크게 향상된다.

```scala
KVcharacters.reduceByKey(addFunc).collect()
```

### 기타 집계 메소드

구조적 API 를 사용하면 훨씬 간단하게 집계를 수행할 수 있다.

### aggregate

이 함수는 null 값이나 집계의 시작값이 필요하며 두 가지 함수를 피라미터로 사용한다.

첫 번째 함수는 파티션 내에서 수행되고 두 번째 함수는 모든 파티션에 걸쳐서 수행된다.

두 함수 모두 시작값을 사용한다.

```scala
nums.aggregate(0)(maxFunc, addFunc)
```

aggregate 는 최종적으로 드라이버에서 집계를 하기 때문에 성능에 약간의 영향이 있다.

익스큐터의 결과가 너무 클 경우에 드라이버에서 OutOfMemoryError 가 발생할 수 있다.

aggregate 와 동일한 작업을 수행하지만 다른 처리 과정을 거치는 treeAggregate 도 있다.

이는 드라이버에서 집계하기전에 익스큐터끼리 트리를 구성해서 집계 처리의 일부 하위 과정을 ‘push down’ 방식으로 먼저 수행한다.

이를 통해서 드라이버의 메모리를 모두 소비하는 현상을 막는다.

```scala
val depth = 3
nums.treeAggregate(0)(maxFunc, addFunc, depth)
```

### aggregateByKey

aggregateByKey 함수는 aggregate 와 동일하지만 파티션 대신 키를 기준으로 연산을 수행ㅎ나다.

```scala
KVcharacters.aggregateByKey(0)(addFunc, maxfunc).collect()
```

## cogroup

cogroup 함수는 스칼라를 사용하는 경우 최대 3 개 키-값 형태의 RDD 를 그룹화 해서 각 키를 기준으로 값을 결합한다.

즉 RDD 에 대한 그룹 기반의 조인을 수행한다.

주로 출력 파티션 수나 클러스터에 데이터 분산 방식을 정확하게 제어하기 위해서 사용한다.

```scala
charRDD = distinctChars.map(lambda c: (c, random.random()))
charRDD2 = distinctChars.map(lambda c: (c, random.random()))

charRdd.cogroup(charRDD2).take(5) 
```

## 조인

RDD 는 구조적 API 에서 알아본 것과 거의 동일한 조인 방식을 가지고 있지만 사용자가 많은 부분에 관여해야 한다.

RDD 나 구조적 API 나 모두 기본 형식을 사용하고 조인 하려는 두 개의 RDD 가 기본적으로 필요하다.

때에 따라서 파티션 수나 사용자 정의 파티션 함수를 파라미터로 사용한다.

### 내부 조인

출력 파티션 수를 어떻게 설정하는지 주의 깊게 살펴보자.

```scala
val keyedChars = distinctChars.map(c => (c,new Random().nextDouble()))
val outputPartitions = 10

KVcharacters.join(keyedChars).count()
KVcharacters.join(keyedChars, outputPartitions).count()
```

### zip

마지막 조인 타입은 사실 진짜 조인은 아니지만 두 개의 RDD 를 결합하므로 조인이라고 볼 수 있다.

zip 함수를 사용해 동일한 두 개의 RDD 를 지퍼 (zipper) 잠그듯이 연결할 수 있으며 PairRDD 를 생성한다.

두 개의 RDD 는 동일한 수의 요소와 동일한 수의 파티션을 가져야 한다.

```scala
val numRange = sc.parallelized(0 to 9, 2)
words.zip(numRange).collect()
```

## 파티션 제어하기

RDD 를 사용하면 데이터가 클러스터 전체에 물리적으로 정확히 분산되는 방식을 정의할 수 있다.

이러한 기능을 가진 메소드 중 일부는 구조적 API 에서 사용했던 메소드와 기본적으로 동일하다.

구조적 API 와 가장 큰 차이점은 파티션 함수를 파라미터로 사용할 수 있다는 점이다.

파티션 함수는 보통 사용자 지정 Partitioner 를 의미한다.

### coalesce

coalesce 는 파티션을 재분배할 때 발생하는데이터 셔플을 방지하기 위해 동일한 워커에 존재하는 파티션을 합치는 메소드다.

예를 들어 예제의 words RDD 는 현재 두 개의 파ㅣ티션으로 구성되어 있고 coalesce 메소드를 통해 데이터 셔플링 없이 하나의 파티션으로 합칠 수 있다.

```scala
words.coalesce(1).getNumPartitions // 1 이 출력 
```

### repartition

repartition 메소드를 통해 파티션 수를 늘리거나 줄일 수 있지만 처리 시 노드간의 셔플이 발생할 수 있다.

파티션 수를 늘리면 맵 타입이나 필터 타입의 연산을 수행할 때 병렬 처리의 수준을 높일 수 있다.

```scala
words.repartition(10) // 10 개의 파티션이 생성 
```

### 사용자 정의 파티셔닝

사용자 정의 파티셔닝 (custom partitioning) 은 RDD 를 사용하는 가장 큰 이유 중 하나다.

구조적 API 에서는 사용자 정의 파티셔닝을 파라미터로 사용할 수 없다.

사용자 정의 파티셔닝의 대표적인 예제는 PageRank 이다.

PageRank 는 사용자 정의 파티셔닝을 이용해 클러스터의 데이터 배치 구조를 제어하고 셔플을 회피한다.

사용자 정의 파티셔닝의 유일한 목표는 데이터 치우침 (skew) 현상을 피하고자 클러스터 전체에 걸쳐 데이터를 균등하게 배분하는 것이다.

사용자 정의 파티셔닝을 사용하려면 구조적 API 로 RDD 를 얻고 사용자 파티셔닝을 적용한 후 다시 DataFrame 이나 Dataset 으로 변환하는 것이다.

사용자 정의 파티셔닝을 사용하려면 Partitioner 를 확장한 클래스를 구현해야한다.

주의 할 점은 문제에 대한 충분한 업무 지식을 가지고 ㅣㅇㅆ는 경우에만 사용하는 게 좋다.

단일 값이나 다수 값을 파티셔닝해야 한다면 DataFrame API 를 사용하는 것이 좋다.

예제로보자.

```scala
val df = spark.read.option("header", "true")
	.option("inferSchema", "true")
	.csv("/data/retail-data/all/")

val rdd = df.coalesce(10).rdd 
```

HashPartitioner 와 RangePartitioner 는 RDD API 에서 사용할 수 있는 내장형 파티셔너 이다. 각각 이산형과 연속형 값을 다룰 때 사용한다.

```scala
rdd.map(r => r(6)).take(5).foreach(println)
val keyedRDD = rdd.keyBy(row => row(6).asInstanceOf[Int].toDouble)

keyedRDD.partitionBy(new HashPartitioner(10)).take(10)
```

매우 큰 데이터나 심각하게 치우친 키를 다뤄야 한다면 고급 파티셔닝 기능을 사용해야 한다.

키 치우침은 어떤 키가 다른 키에 비해 아주 많은 데이터를 가지는 현상을 의미한다.

병렬성을 개선하고 실행 과정에서 OutOfMemoryError 를 방지할 수 있도록 키를 최대한 분할해야한다.

키가 특정 형태를 띠는 경우에만 키를 분할해야한다.

예를 들어 데이터셋에 항상 분석 작업을 어렵게 만드는 두 명의 고객 정보가 있다면

다른 고객의 정보와 두 고객의 정보를 분리해야한다.

물론 다른 고객의 정보를 하나의 그룹으로 묶어서 처리할 수 있다.

하지만 두 고객의 정보와 관련된 데이터가 너무 많아 치우침이 심하게 발생한다면 나누어서 처리해야 할 수 있다.

예제로보자.

```scala
class DomainPartitioner extends Partitioner {
	def numPartitions = 3 
	def getPartition(key: Any): Int = {
		val customerId = key.asInstanceOf[Double].toInt
		if (customerId == 17850.0 || customerId = 12583.0) {
			return 0 
		}
		return new java.util.Random().nextInt(2) + 1 
	}
}

keyedRDD.
	partitionBy(new DomainPartitioner).map(_._1).glom()
	.map(_.toSet.toSeq.length)
	.take(5) 
```

## 사용자 정의 직렬화

Kryo 직렬화를 통해서 직렬화 성늘을 높일 수 있다.

병렬화 대상인 모든 객체는 직렬화가 가능해야한다.

SparkConf 를 사용해 잡을 초기화하는 시점에서 spark.serializer 속성값을 org.apache.spark.serializer.KryoSerializer 를 설정해서 Kryo 를 사용할 수 있다.

spark.serializer 설정으로 워커 노드 간 데이터 셔플링과 RDD 를 직렬화해 디스크에 저장하는 용도로 사용할 Serializer 를 지정할 수 있다.