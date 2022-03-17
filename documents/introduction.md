# Spark Introduction

## Spark 의 정의

Spark 의 정의는 다음과 같다.

> Apache Spark™ is a multi-language engine for executing data engineering, data science, and machine learning on single-node machines or clusters.
>
- Spark 은 Scala, Java, Python, R 과 같은 여러 언어를 지원하며 데이터 처리나 ML 에 사용 가능한 엔진이다. (딥러닝 쪽은 아직은 그렇게 발전하지는 않았다고 한다.
- single-node 로도 처리할 수 있지만 클러스터 환경에서 컴퓨팅 파워를 모아서 실행하는 것도 가능하다. (그래서 클러스터 환경에서 리소스를 할당해주는 클러스터 매니저가 필요하다. 물론 내부적으로도 있다.)

### Key features

Spark 의 핵심 기능은 다음과 같다고 한다.

- **Batch/Streaming data**: Unify the processing of your data in batches and real-time streaming, using your preferred language
- **SQL analytics**: Execute fast, distributed ANSI SQL Queries for dashboarding and ad-hoc reporing run faster than most data warehouses
- **Data science at scale**: Perform Exploratory Data Analysis (EDA) on petabyte-scale data without having to resort to downsampling
- **Machine learning**: Train machine learning algorithms on a laptop and use the same code to scale to fault-tolerant clusters of thousands of machines
- **Graph processing**: GraphX is a new component in Spark for graphs and graph-parallel computation.

## Spark Architecture

Apache Spark 은 다음과 같은 layered architecture 로 이뤄진다.

![](./images/spark%20architecture.png)

- Spark 은 Driver 프로세스와 Executor 프로세스 그리고 클러스터 매니저로 이뤄진다.
- Driver 프로세스는 클러스터에 있는 노드 중 하나이며 Spark 어플리케이션의 main() 함수를 실행하는 역할을 한다. 즉 사용자 코드를 시작하는 역할을 한다.
    - 사용자 코드를 시작하는 일을 Job 을 만든다고 생각하면 된다. Job 을 만들 때 클러스터 매니저와 통신해서 리소스를 할당해서 만든다.
- Executor 프로세스는 드라이버 프로세스가 할당한 작업을 수행한다. 이 수행이 끝나면 드라이버 프로세스에게 다시 보고하며 마무리 된다.
    - 즉 드라이버가 만든 Job 을 실행해주는 역할을 하고 결과적으로 작업이 수행되면 디스크나 메모리에 저장하는 역할을 한다.
    - Spark 은 어떠한 저장소에도 의존하지 않고 영구 저장소 역할을 수행하지도 않는다. 그래서 Amazon S3 이나 Hadoop, Azure Storage, Apache Kafka 등에다가 저장하는 것도 가능하다.
- 스파크는 한 대의 컴퓨팅 머신을 이용할 수도 있지만 여러 컴퓨터의 자원을 모아서 사용한다. 이렇게 하기 위해서 클러스터 매니저가 필요한데 컴퓨팅 리소스를 할당하고 관리해주는 역할을 한다.
- 드라이버 프로세스는 SparkSession (이전에는 SparkContext 라고도 불렸다) 을 통해서 클러스터와 연결하고 Spark 의 여러 API 를 사용하는게 가능하다. 스파크를 사용할려면 첫 진입점으로 SparkSession 을 만들어야 한다.

## Spark 의 여러 API 들

![](./images/spark%20api.png)

- Spark 을 사용하는 방법은 저수준 API 인 RDD 와 그보다 고수준인 DataFrame 과 DataSet 이 있다.
- RDD 는 Job 을 실행할 때 처리하는 데이터로 클러스터 간에 분할되며 변경할 수 없다. 그리고 여러 노드에서 병렬로 작동된다.
- Spark 에선 RDD 를 처리할 때는 두 가지 방식이 있다.
    - **transformation:** lazy operation 이다. action 이 호출될 때까지 데이터를 일시적으로 보유한다. RDD 는 변경할 수 없으므로 transformation 은 새로운 RDD 를 생성한다. 예로는 map, groupby, reduceby, sortby, filter, join, union 등이 있다.
    - **action:** action 을 통해서 클러스터에서 실행한 job 은 스파크 드라이버로 돌아간다. action 의 예로는 Reduce, Collect, Take, First, SaveAsTextfile, saveAsSequecneFile, foreach 등이 있다.
- DataFrame 은스파크에서 가장 많이 사용하는 데이터 구조다. 행과 열이 있는 데이터 테이블과 같으므로 SQL 사용자가 많이 사용한다. 비정형 데이터의 경우 RDD, 정형 데이터의 경우 데이터 프레임을 사용한다고 알면 된다.
- Dataset 은 자바나 스칼라와 같이 정적 타입을 지원하기 위한 스파크의 구조적 API 이다. DataFrame 과 비슷한데 DataFrame 은 테이블형 데이터를 보관할 수 있는 Row 타입의 객체로 구성되었다. Dataset 은 여기에다가 클래스에 할당할 수 있도록 해서 타입 안정성을 지원했다. 그래서 컴파일타임의 에러를 발견할 수 있다.