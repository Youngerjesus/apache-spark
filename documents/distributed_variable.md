# 분산형 공유 변수

스파크 저수준 API 에서는 RDD 말고도 ‘분산형 공유 변수’ 가 있다.

분산형 공유 변수에는 브로드캐스트 변수와 어큐뮬레이터 라는 두 개의 타입이 있다.

이는 RDD 나 DataFrame 을 다루는 map 함수에서 이 변수를 사용할 수 있다.

특히 어큐뮬레이터를 사용하면 모든 테스크의 데이터를 공유 결과 변수에 추가할 수 있다.

예를 들어 Job 의 입력 레코드를 파싱하면서 얼마나 많은 오류가 발생했는지 확인하는 카운터를 구현할 수 있다.

반면 브로드캐스트 변수를 사용하면 모든 워커 노드에 큰 값을 저장하므로 재전송 없이 많은 스파크 액션에서 재사용 할 수 있다.

## 브로드캐스트 변수

브로드캐스트 변수는 변하지 않는 값을 클로저 (closure) 함수의 변수로 캡슐화하지 않고 클러스터에서 효율적으로 공유하는 방법을 제공한다.

테스크에서 드라이버 노드의 변수를 사용할 때는 클로저 함수 내부에서 단순하게 참조하는방법을 사용한다.

하지만 이는 비효율적이다.

특히 룩업 테이블이나 머신러닝 모델 같은 큰 변수를 사용하는 경우에 더 비효율적이다.

이유는 클로저 함수에서 변수를 사용할 때 우커 노드에서 여러번 역직렬화가 일어나기 때문이다.

게다가 여러 스파크 액션과 동일한 변수를 사용하면 잡을 실행할 때마다 워커로 큰 변수를 재전송한다.

이런 상황에서는 브로드캐스트 변수를 사용한다.

브로드 캐스트 변수는 모든 태스크마다 직렬화 하지않고 모든 머신에 캐시하는 불변성 공유 변수이다.

예로 익스큐터 메모리 크기에 맞는 조회용 테이블을 전달하고 함수에서 사용하는 것이 대표적이다.

예를 들어 다음 예제처럼 구조체를 스파크에 브로드캐스트해서 클러스터의 모든 노드에 지연 처리 방식으로 복제할 수 있다.

이를 통해 이제 직렬화 역직렬화에 대한 부하를 크게 줄일 수 있다.

```scala
val supplementalData = Map("Spark" -> 1000, "Definitive" -> 2000, "Big"  -> -300, "Simple" -> 100
```

즉 브로드캐스트 변수를 사용한 방식이 클로저를 사용한 방식보다 훨씬 효율적이다.

## 어큐뮬레이터

어큐뮬레이터는 트랜스포메이션 내부의 다양한 값을 갱신하는 데 사용한다.

그리고 내고장성을 보장하면서 드라이버에 값을 전달할 수 있다.

어큐뮬레이터는 스파크 클러스터에서 로우 단위로 안전하게 값을 갱신할 수 있는 변경 가능한 변수를 제공한다.

그리고 디버깅용이나 저수준 집계 생성용으로 사요할 수 있다.

예로 파티션별로 특정 변수의 값을 추적하는 용도로 사용할 수 있다.

어큐뮬레이터의 값은 액션을 처리하는 과정에서만 갱신된다.

예제를 보자. 여기서는 RDD API 가 아닌 Dataset API 를 이용했다.

```scala
val flights = spark.read
	.parquet("/data/fligh-data/parquest/2010-summary.parquet")
	.as[Flight]

val accChina = new LongAccumulator
val acc = spark.sparkContext.register(accChina, "China") 

flights.foreach(flight_row => accChinaFunc(flgiht_row))

def accChinaFunc(flight_row: Flight) = {
	val destination = flight_row.DEST_COUTRY_NAME
	val origin = flight_row.ORIGIN_COUTRY_NAME

	if (destination == "CHINA") {
		accChina.add(flight_row.count.toLong)
	}
	if (origin == "CHINA") {
		accChina.add(flight_row.count.toLong
	}
}
```