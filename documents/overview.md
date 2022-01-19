# Apache Spark 

Spark 의 정의는 레퍼런스에는 다음과 같이 적혀있다.

`A fast and general engine for large-scale data processing`

Spark 은 Hadoop 의 MapReduce 를 대체한다. 

- 100 배 정도 더 빠르기 떄문에.
- DAG Engine (directed acyclic graph) 때문에 더 빠르다. 
  - 드라이버 매니저에 전달한 workflow 를 최적화 해주는 역할을 한다.  

Spark Datasets, DataFrame 은 SQL 과 비슷하다. 
- Spark SQL 을 통해서 다루는 것도 가능하다. 

SQL 뿐 아니라 Lower-level API 를 다루고 싶다면 RDD (Resilient Distributed Dataset) 을 통해 가능하다.

실시간 데이터 수집 및 처리를 위한 Spark Streaming 도 지원. 