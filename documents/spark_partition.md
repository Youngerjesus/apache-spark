# Spark 과 Partition

Paritition 은 RDD 나 Dataset 을 이루는 최소 단위라고 생각하면 된다.

그리고 하나의 Task 가 이 하나의 Partition 을 처리하는 개념.

- Task 는 Job 의 개념이라면 Partition 은 Data 의 개념이다.

Partition 을 만들 때는 CPU Core 수에 맞춰서 만들면 가장 병렬성이 좋다. 그치만 Partition 은 모두 메모리에 올라가니까 이것도 생각해야함.

Partition 의 종류는 다음과 같다.

- Input Partitoin
- Output Partition
- Shuffle Partition

Input Partition 과 관련된 설정의 경우 ``spark.sql.files.maxpartitionBytes`` 가 있다.

- 처음 파일을 읽을 때 파일 하나당 파티션 하나가 매칭되는데 파일 크기를 어느정도로 정할 것인지를 설정하는 옵션이라고 생각하면 된다. 기본 값은 128 MB 이다.

Output Partition 과 관련된 설정의 경우 ``df.repartition(cnt)`` 와 ``df.coalesce(cnt)`` 가 있다.

- Output Partition 으로 지정된 개수만큼 마지막 저장할 파일 수를 지정한다.
- 예로 50 으로 저장하면 50 개로 저장됨. 주로 ``spark.sql.shuffle.partitions` 설정에 따라서 파일 수가 지정이 되는데 이때 파일의 크기를 늘리기위해 repartition 과 coalesce 를 이용해서 파티션 수를 줄여서 파일 크기를 늘리는 방법도 있다. (파일 : Partition 은 1:1)

Shuffle Partition 의 경우에는 관련 설정으로 ``spark.sql.shuffle.partitions`` rk dlTek.

주로 join, groupby 연산을 수행할 때 Shuffle partition 이 쓰인다.

이 설정 값은 CPU Core 의 수에 맞게 설정하라곤 하지만 partition 의 크기에 맞춰서 설정하는 것도 중요하다.

이 Partition 의 크기가 큰데 메모리가 부족하다면 Shuffle Spill 이 발생해서 스토로지 영역을 사용한다.

- 파티션이 작다면 거기안에 모여있는 데이터는 많아질 것.