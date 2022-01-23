# RDD Basic 

스파크에서는 대부분에서는 고수준 API 인 구조적 API 를 사용해서 문제를 해결할 수 있지만 모든 문제를 해결할 수 있는건 아니다. 그래서 저수준의 API 를 알아야 한다.

저수준의 API 는 다음과 같다.

- RDD
- SparkContext
- Accumulator
- Broadcast Variable

## 저수준 API 란

스파크에는 두 종류의 저수준 API 가 있다.

하나는 데이터를 처리하기 위한 RDD 가 있고, 다른 하나는 브로드캐스트 변수와 어큐뮬레이터 처럼 분산형 공유 변수를 배포하고 다루기 위한 API 가 있다.

SparkContext 를 통해서 저수준 API 를 사용할 수 있다.

## RDD 개요

RDD 는 스파크 1.x 버전에서는 핵심이었다. 스파크 2.x 에서도 사용할 순 있지만 구조적 API 를 더 많이 사용한다.

그치만 결국 DataFrame 이나 Dataset 도 RDD 로 컴파일 되기도 하고 스파크 UI 에서도 RDD 단위로 잡이 수행됨을 알 수 있다. 즉 RDD 는 알고 있어야 한다.

RDD 는 불변성을 가지며 병렬로 처리할 수 있는 파티셔닝된 레코드의 모음이다.

DataFrame 의 레코드는 각 스키마를 가지는 반면에 RDD 는 그저 자바, 스칼라의 객체일 뿐이다.

RDD 의 모든 레코드는 자바나 파이썬의 객체이므로 이를 완벽하게 제어하는게 가능하다.

하지만 구조적 API 와는 다르게 레코드의 내부 구조를 스파크가 아는 형태가 아니므로 최적화를 제공해줄 수 없다. 구조적 API 는 자동으로 데이터를 최적화하고 압축된 바이너리 포맷으로 저장하는 반면에.

### RDD 유형

사용자는 두 가지 타입의 RDD 를 만드는게 가능하다.

- 하나는 제네릭 RDD 타입이고 다른 하나는 집계가 가능한 키-값 RDD 이다.

둘 다 객체의 컬렉션을 표현하지만 키-값 RDD 는 특수연산 뿐 아니라 키를 이용해서 사용자 지정 파티셔닝 개념을 가지고 있다.

RDD 는 내부적으로 다음 다섯 가지 주요 속성을 가지고 있다.

- 파티션의 목록
- 각 조각을 연산하는 함수
- 다른 RDD 와의 의존성 목록
- 부가적으로 키-값 RDD 를 위한 Partitioner
- 부가적으로 각 조각을 연산하기 위한 기본 위치 목록

RDD 역시 스파크 프로그래밍 패러다임을 그대로 가지고 있어서 트랜스포메이션과 액션을 제공한다.

DataFrame 과 Dataset 의 트랜스포메이션, 액션과 동일한 방식으로 제공하지만 RDD 에는 ‘Row’ 라는 개념은 없다.

개별 레코드는 자바, 스칼라의 객체일 뿐이다.

그리고 RDD 를 사용할 땐 파이썬을 가급적이면 사용하지 않는게 좋다.성능이 안나와서 그렇다.

- 파이썬 객체 → 직렬화해서 JVM 에게 전달 → 처리 → 역직렬화 → 다시 파이썬에게 전달. 이 과정에서 직렬화 역직렬화 작업의 오버헤드가 있기 떄문에 성ㄴ능이 나오지 않는다.

그럼 RDD 는 언제 사용할까?

DataFrame 이 있기 때문에 일반적인 처리는 DataFrame 이나 Dataset 으로 처리하는게 좋다. 다만 세밀한 제어가 필요할 땐 RDD 를 사용하자.

### RDD 생성하기

RDD 를 가장 얻기 쉬운 방법은 기존에 DataFrame 이나 Dataset 을 이용하고 있다면 그걸 통해서 얻는 것이다.

예로 Dataset[T] 를 RDD 로 변환하면 데이터 타입 T 를 가진 RDD 를 얻을 수 있다.

```scala
spark.range(10).toDF().rdd.map(rowObject => rowObject.getLong(0))
```

RDD 를 사용해서 DataFrame 이나 Dataset 을 생성할 때는 toDF() 메소드를 호출하면 된다.

- 이게 가능한 이유는 rdd 메소드는 Row 타입을 가진 RDD 를 생성하는데 Row 타입은 스파크가 구조적 API 에서 데이터를 표현하는 데 사용하는 내부 카탈리스트 포맷이기 때문이다.

## RDD 다루기

- RDD 를 다루는 건 DataFrame 을 다루는 것과 비슷하다. 하지만 DataFrame 에 비해서 연산을 단순화 하는 함수나 집계하는 함수가 부족해서 직접 사용자가 정의해야한다.

## 트랜스포메이션

대부분 RDD 트랜스포메이션은 구조적 API 에서 사용할 수 있는 기능들을 가지고 있다.

그리고 DataFrame 이나 Dataset 과 동일하게 RDD 에 트랜스포메이션을 적용해서 새로운 RDD 를 사용하는게 가능하다.

### distinct

RDD 의 distinct 메소드를 호출하면 RDD 에서 중복된 데이터를 제거한다.

```scala
words.distinct().count()
```

### filter

필터링은 SQL 의 where 조건절을 생성하는 것과 비슷하다. RDD 의 레코드를 모두 확인하고 조건 함수를 만족하는 레코드만 반환한다.

filter 로 동작하는 조건 함수는 boolean 타입을 리턴해야한다.

```scala
words.filter(word => startsWithS(word)).collect()
```

### map

주어진 입력을 원하는 값으로 반환하는 함수가 map 이다.

예제에서ㅓ는 현재 단어를 ‘단어’, ‘단어의 시작 문자’, ‘첫 문자가 S인지 아닌지’ 로 바꿨다.

```scala
val words2 = words.map(word => (word, word[0], startWithS(word))
```

### flatMap

flatMap 은 단일 로우를 flat 해서 여러 로우로 변환하는 작업을 한다.

예로 단어를 문자 집합으로 변환시키는게 flatMap 이다.

```scala
val charSet = words.flatMap(word ⇒ word.toSeq)
```

### sortBy

RDD 를 정렬하려면 sortBy 메소드를 사용하면 된다.

sortBy 는 RDD 에서 데이터를 추출한 다음 그 값을 기준으로 정렬한다.

다음 예제는 RDD 에서 단어 길이가 긴 것부터 짧은 순으로 정렬하는 것이다.

```scala
words.sortBy(word => word.length() * -1) 
```

### randomSplit

randomSplit 은 RDD 를 임의로 분할해서 RDD 배열을 만들 때 사용한다.

## 액션

액션은 데이터를 드라이버로 모우거나 외부 데이터 소스로 내보낼 때 사용한다.

### reduce

RDD 의 모든 값을 모아서 하나로 만들려면 reduce() 메소드를 사용하면 된다.

reduce() 에서는 한 개의 누적값과 집합에서 다음 요소의 값을 인자로 받는다.

reduce() 를 통해서 두 개의 입력값을 하나로 줄이도록 하는게 가능하다.

다음 예는 가장 긴 단어를 찾는 걸 reduce() 로 해결한 것이다.

```scala
def wordLengthReducer(leftWord:String, rightWord:String): String = {
	if (leftWord.length > rightWord.length) return leftWord

	return rightWord
}

words.reduce(wordLengthReducer)
```

### count

count 함수를 사용하면 RDD 의 전체 로우 수를 알 수 있다.

```scala
words.count() 
```

### countApprox

이 함수는 count() 함수로 나올 결과를 제한 시간 내에 계산하는 함수다. 꽤 정교한 편이다.

countApprox 에서는 신뢰도와 제한시간을 설정해놓고 호출하는데 꽤 정확하다.

```scala
val confidence = 0.95
val timeoutMillseconds = 400
words.countApprox(timeoutMillliseconds, confidence)
```

### countByValue

이 메소드는 RDD 값의 개수를 구하는데 결과 데이터셋을 드라이버의 메모리로 읽어들여서 처리한다.

즉 이 메소드를 사용하면 Executor 의 연산 결과라 드라이버 메모리에 모두 적재된다. 그러므로 결과가 작을 때만 사용하자.

```scala
words.countBtValue()
```

### first

first 메소드는 데이터셋의 첫 번째 값을 반환한다.

```scala
words.first()
```

### min 와 max

max 와 min 메소드는 각각 최댓값과 최솟값을 반환한다.

### take

RDD 에서 가져올 값의 개수를 파라미터로 전달받아서 그 개수만큼만 가지고 온다.

## 캐싱

RDD 캐싱에도 DataFrame 이나 Dataset 캐싱과 동일한 원칙이 적용된다.

RDD 를 캐시하거나 저장 (persist) 할 수 있다.

캐시와 저장은 메모리에 있는 데이터만을 대상으로 한다.

setName() 를 통해 캐시된 RDD 에 이름을 지정할 수 있다.

```scala
words.cache()
```

저장소 수준은 싱글톤 객체인 org.apache.spark.storage.StorageLevel 의 속성 (메모리, 디스크, 또는 둘의 조합 그리고 off-heap 이 있다.) 중 하나로 지정할 수 있다.

```scala
words.getStorageLevel
```

## 체크포인팅

DataFrame 에서 사용할 수 없는 기능 중 하나가 체크포인팅 (Checkpointing) 이다.

체크포인팅은 RDD 를 디스크에 저장하는 방식이다.

나중에 저장된 RDD 를 참조할 때 원본 데이터소스를 다시 계산해서 RDD 를 생성하지 않고 디스크에 저장된 중간 결과 파티션을 참조한다.

이런 동작은 메모리에 저장하지 않고 디스크에 저장한다는 사실만 제외하면 캐싱과 유사하다.

이 기능은 반복적인 연산 수행 시 매우 유용하다.

```scala
spark.sparkContext.setCheckpointDir("/some/path/for/checkpointing")
words.checkpoint()
```

## mapPartitions

스파키는 실제 코드를 실행할 때 파티션 단위로 동작한다.

map 함수에서 반환하는 RDD 의 진짜 형태는 사실 MapPartitionsRDD 이다.

map 은 이것의 alias 일 뿐이다.

mapPartitions 는 개별 파티션에 대해서 map 연산을 수행할 수 있다.

그 이유는 클러스터에서 물리적인 단위로 개별 파티션을 처리하기 때문이다.

다음은 데이터의 모든 파티션에 ‘1’ 값을 생성하고 파티션 수를 세어서 합산한다.

```scala
words.mapPartitions(part => Iterator[Int](1)).sum()  
```