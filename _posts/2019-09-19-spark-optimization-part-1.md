---
title: "Spark 성능 최적화 및 튜닝 방법 - Part 1"
date: 2019-09-19 21:45 +0900
categories: Data
---


최근에 Spark를 사용하면서 각종 High level API (Dataset, Dataframe) 와 어떻게 하면 Spark를 조금이라도 빠르게 쓸 수 있을지에 대한
고민을 하기 시작했는데요. Spark를 AWS EMR을 이용해서 돌리고 있고, EMR은 사용한 시간만큼 돈을 내는 구조기 때문에 Spark app이 빨리 끝나면
끝날 수록 돈을 절약할 수 있기 때문입니다. (그리고 빠르면 빠를 수록 엔지니어로서의 희열도 느껴지기 때문에...) 그리하여, `어떻게 하면 최적의 Spark 코드를 짤 수 있을까?` 라는 목표로
Spark API에 대한 공부를 시작했습니다.

아직 최적의 Spark 코드를 짤 수 있는 방법을 알아낸 것은 아니지만, 적어도 다음과 같은 사항을 지키면 `최악의 Spark 코드`는 되지 않겠다는
생각을 했습니다. `Part 1`에서는 프로그래머가 코드 단에서 최적화할 수 있는 방법들을 소개합니다.

> [참고한 블로그 (영문)](https://michalsenkyr.github.io/2018/01/spark-performance)

### 1. `groupByKey` 와 `reduceByKey`
(이건 너무 유명한 팁이지만) `reduceByKey`로 해결할 수 있는 문제 상황에서는 무조건 `reduceByKey`를 사용해야 합니다. `groupByKey`를 쓰게 되면,
Spark에서 가장 기피해야할 (하지만 어쩔 수 없는) `Data shuffling`이 모든 node 사이에서 일어나게 됩니다. 물론 `reduceByKey`를 사용해도 동일하게
`shuffling`은 일어나지만, 두 함수의 가장 큰 차이점은 `reduceByKey`는 `shuffle` 하기 전에 먼저 `reduce` 연산을 수행해서 네트워크를 타는 데이터를 현저히 줄여줍니다.
그래서 가급적이면 `reduceByKey`나 `aggregateByKey` 등 `shuffling` 이전에 데이터 크기를 줄여줄 수 있는 함수를 먼저 고려해야 합니다. 똑같은 `Wide transformation` 함수라도
성능 차이가 엄청납니다.

#### Dataset API에서 `groupBy.reduceGroups` 사용하기 (Spark Dataset API에 `reduceByKey`가 없는 이유)
하지만 Dataset API에는 `reduceByKey` 연산이 존재하지 않는데요. 대신 `groupBy.reduceGroups` 형태로 존재합니다. RDD와 달리
Dataset API부터는 `groupBy`나 `groupByKey`를 호출한 이후에 어떤 연산을 수행하냐에 따라 Spark에서 자동으로 최적화를 진행해줍니다. 즉, `groupBy.reduceGroups`의 경우에는
맨 처음에 `groupBy`를 적용하는 것처럼 보이지만, 실제로는 `reduceByKey` 처럼 동작합니다. 그래도 아직 `reduceByKey` 보다는 1.x배 느리다는 벤치마크 결과가 있습니다.

[groupByKey와 reduceByKey에 대해 더 자세히 알고 싶으시다면 이 글을 추천합니다!](https://www.ridicorp.com/blog/2018/10/04/spark-rdd-groupby/)

### 2. Partitioning
Spark cluster와 같은 병렬 환경에서는, 데이터를 알맞게 쪼개어주는 것이 매우 중요합니다. 그래야 각 executor node가 놀지 않고 일을 할 수 있으니까요.
잘 쪼개지지 않은 데이터를 가지고 일을 시키면, 특정 node에게만 일이 몰리는 현상이 발생할 수 있습니다. 이를 데이터가 `skew` 되었다고 하죠.
프로그래머가 통제할 수 있는 상황에서는 `coalesce`나 `repartition` 함수를 통해 partition 개수를 적절히 설정해줄 수 있지만, 프로그래머가 통제할 수 없는 상황도 있습니다. 바로 `join` 등과 같이
`imply shuffling`이 일어날 때인데요. 이때는 Spark 설정값인 `spark.sql.shuffle.partitions` 값으로 partition 개수가 정해집니다. 그래서 `join` 연산 등이 빈번하게 일어나는 job의 경우에는
미리 해당 설정값을 적절히 조절해주는 것으로 적당한 partition 개수를 유지할 수 있습니다.

#### `coalesce` vs `repartition`
여기서 하나 주의해야 할 사실은, `repartition` 함수는 `shuffling`을 유발한다는 것입니다. 왜냐하면 `repartition` 자체가 전체 데이터를 node 사이에 균등하게 분배해주는 것이므로, 당연히
`shuffle`이 일어날 수밖에 없겠죠? `coalesce` 함수를 사용하게 되면 partition 개수를 늘릴 수 없는 제약이 있는 대신에, `shuffle`을 유발하지 않고도 데이터를 분배할 수 있습니다.
(그런데 어떤 원리로 `coalesce`가 `shuffle` 없이 데이터를 균등 분배할 수 있는지는 모르겠네요. 시간이 되면 살펴봐야겠습니다)
```scala
def coalesce(numPartitions: Int): Dataset[T] = withTypedPlan {
    Repartition(numPartitions, shuffle = false, planWithBarrier)
}

def repartition(numPartitions: Int): Dataset[T] = withTypedPlan {
    Repartition(numPartitions, shuffle = true, planWithBarrier)
}
```
> 실제 `coalesce`와 `repartition`은 모두 `Repartition` 함수를 호출하게 되어있습니다. 두 함수의 차이는 shuffle 여부밖에 없군요.

### 3. Serializer 선택
Scala의 가장 큰 장점 중 하나는 바로 `case class` 라고 생각합니다. case class와 Spark를 결합하면 큰 노력없이 type strict한 코드를 작성할 수 있는데요.
문제는 사용자가 `case class`를 사용하면 Spark가 각 object를 node 사이에 분배할 때 serialization/deserialization이 일어나게 됩니다. (`SerDe` 입니다.)
Spark 2.x 버전을 기준으로, Spark는 두 가지 형태의 serializer를 지원하는데요. 기본값으로 설정되어 있는 `Java serializer`와 성능이 월등히 개선된 `Kyro serializer`가 그 주인공입니다.
어떤 이유에선지 `Kyro`가 성능이 훨씬 좋음에도 불구하고 기본 serializer로 설정되어 있지 않아, 사용자가 다음 설정을 통해 `Kyro`를 사용하도록 만들어줘야 합니다.

> spark.serializer "org.apache.spark.serializer.KryoSerializer"

Spark 2.x에서는 명시적으로 설정해주지 않아도 몇 가지 기본적인 연산 (`shuffling with primitive types`) 등에 대해서는 자동으로 `Kyro`를 사용하고 있습니다. 그래도 모든 연산에 적용될 수 있도록 설정해주는 편이 훨씬 좋겠죠?
참고로, `Kyro`로 `SerDe`를 수행하다 실패하는 경우에는 자동으로 `Java serializer`로 fallback 되니, 안심하고 사용하셔도 됩니다. 벤치마크에 따르면, `Kyro`가 기존 `Java serializer`보다 약 10배 빠르다고 합니다.

### 4. High-level API 사용하기
Spark 2.x 부터는 Dataset API를 사용하는 것이 권장됩니다. 물론 Dataset도 내부 뼈대는 여전히 RDD지만, 다양한 최적화 (`Catalyst optimization` 등) 기법과 훨씬 더 강력한 인터페이스를 포함하고 있습니다.
예를 들어, 시간이 많이 걸리는 `join` 연산을 수행할 때 High-level API를 사용하면 가능한 경우에 자동으로 `Broadcast join` 등으로 바꿔 `shuffle`이 일어나지 않게 해주는 최적화가 이루어집니다.
그래서 가급적이면 저도 무조건 Dataset이나 Dataframe을 이용해서 Spark 코드를 짜려고 노력하고 있습니다.

### 5. Closure serialization
다음 코드를 실행하면 어떤 일이 일어날까요?
```scala
val factor = config.multiplicationFactor
rdd.map(_ * factor)
```

`config`로 부터 특정 상수값을 뽑아서 map에 넘겨주고 있습니다. 이 코드를 실행하면, 우리의 의도와는 다르게 `config` 객체 전부가 `SerDe` 되어 온 node를 돌아다니게 됩니다. 이런 경우에는 object가 크면 클수록 손해를 보게 되겠죠?
내가 원하는 특정 값만 각 node들이 가지게 해주고 싶다면, `Broadcast variable`을 이용하면 됩니다.

```scala
val broadcastedFactor = sc.broadcast(config.multiplicationFactor)
rdd.map(_ * broadcastedFactor)
```

이렇게 하면, 모든 node들이 broadcast된 상수값을 가지고 있을 수 있어 불필요한 `SerDe`가 일어나지 않고 최적의 연산을 수행하게 됩니다.

다음 `Part 2`에서는 코드 바깥에서 Spark를 최적화할 수 있는 방법들을 알아보겠습니다. (Spark parameter tuning, 최적의 cluster 크기/개수 선택 방법, AWS EMR 환경에서 Spark 최적화 하는 방법 등)
