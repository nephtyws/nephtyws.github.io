---
title: "Spark 3.0에 새로 추가된 기능 소개 및 설명)"
date: 2020-07-02 00:30 +0900
categories: Data
---

[Spark 3.0.0](https://spark.apache.org/releases/spark-release-3-0-0.html)이 6월 18일에 출시되었습니다. 정말 오랜만의 major update인 만큼 다양한 feature들이 Spark에 추가되었는데요.
1.x에서 2.x으로 넘어올 때 Dataset API, Catalyst Optimizer 등이 추가되었던 게 벌써 엊그제같은데 벌써 3 버전이 나오다니 감회가 새롭네요. 그만큼 더 공부해야 할 것들도 늘어나겠죠...  

Spark 3.0의 ticket list를 보고 제가 중요하다고 생각하는 변경점/추가된 기능에 대해서 간략히 정리해보았습니다!

### Version Update
- [[SPARK-24417] Build and Run Spark on JDK11](https://issues.apache.org/jira/browse/SPARK-24417)
- [[SPARK-23534] Spark run on Hadoop 3.0.0](https://issues.apache.org/jira/browse/SPARK-23534)
- [[SPARK-26132] Remove support for Scala 2.11 in Spark 3.0.0](https://issues.apache.org/jira/browse/SPARK-26132)
- [[SPARK-30968] Upgrade aws-java-sdk-sts to 1.11.655](https://issues.apache.org/jira/browse/SPARK-30968)
- [[SPARK-30695] Upgrade Apache ORC to 1.5.9](https://issues.apache.org/jira/browse/SPARK-30695)

하나같이 주목할만한 변화들입니다. JDK 11로 인해서 JVM 자체의 성능 향상 (+GC) 을 기대할 수 있게 됐고, Hadoop이 개선됨에 따라 S3에 접속할 때 S3A 등의 최신 conncetor를 사용할 수 있고,
Scala 2.11이 제거됨에 따라 backward compatibility를 고려하지 않아도 되니 아주 사소하면서 동시에 아주 중요한 업데이트들이 이루어진 것 같습니다! 특히 ORC도 최신 버전으로 업데이트 되었다니 기쁘네요 (저는 main format으로 ORC를 사용하기 때문에... ㅎㅎ)

### Major features
- [[SPARK-24615] SPIP: Accelerator-aware task scheduling for Spark](https://issues.apache.org/jira/browse/SPARK-24615)
  - Spark가 GPU를 지원하기 위한 초석으로, standalone, YARN, Kubernetes를 cluster scheduler로 사용할 때 GPU resource도 잘 할당할 수 있도록 도와주는 프로젝트입니다. 이번에 Nvidia에서
  [Making Spark Fly: NVIDIA Accelerates World’s Most Popular Data Analytics Platform](https://blogs.nvidia.com/blog/2020/06/24/apache-spark-gpu-acceleration/) 와 같이 ML이 아닌 상황 (Dataset, Spark SQL 등) 에서도 GPU를 활용할 수 있도록
  하는 프로젝트들을 진행하고 있던데, GPU가 점점 Spark와 같은 분산 처리 프레임워크로도 들어오고 있군요!
- [[SPARK-31412] New Adaptive Query Execution in Spark SQL](https://issues.apache.org/jira/browse/SPARK-31412)
  - AQE는 말 그대로 query plan을 생성할 때 최적의 plan을 짤 수 있도록 optimization 해주는 기법들의 통칭이라고 생각하시면 됩니다. 주로 RBO나 CBO를 적용해서 rule에 걸리는 것들은 rule-based로 치환하고,
  runtime 이전의 statistics를 가지고 cost를 비교해서 cost가 적은 plan을 선택하는 CBO가 query optimization의 큰 줄기들인데요. Spark에는 이미 여러 rule과 CBO option들이 적용되어 있는데 이번에 좀 더 개선된 것 같습니다.
  특히 [[SPARK-29544] Optimize skewed join at runtime with new Adaptive Execution](https://issues.apache.org/jira/browse/SPARK-29544) 에서 볼 수 있듯이 skew join에 관한 optimization이 많이 들어갔습니다. skew join은
  생각보다 실무에서 많이 접할 수 있는 (e.g. 두 데이터를 join하는 데 데이터 양이 너무 차이난다거나, 전체 데이터에 NULL 같은 게 대다수여서 distribute가 잘 안 된다거나) 문제고, 이를 해결하기 위해선 사용자가
  데이터의 분포를 정확히 알고 수동으로 분배해주는 전략이 필요했는데, 이걸 자동으로 해준다니 대단하네요!
- [[SPARK-11150] Dynamic partition pruning](https://issues.apache.org/jira/browse/SPARK-11150)
  - 1만 번대 ticket인 걸로 보아서 굉장히 오래된 ticket인 것 같습니다. 주요 골자는 compile time에 알 수 있는 static partitioning과는 달리 (e.g. join 조건이 `t1.foo = t2.bar AND t2.bar = 1` 처럼 주어지면 `t1.foo`가 `1`이라는 것을 쉽게 유추할 수 있습니다)
  `SELECT * FROM dim_table JOIN fact_table ON (dim_table.partcol = fact_table.partcol) WHERE dim_table.othercol > 10` 이런 query처럼 `partcol`을 compile time에 알 수 없는 상황이 오면 전체 데이터를 다 읽을 수밖에 없습니다. 이런 상황에
  dynamic partitioning을 runtime에 해서 `partcol`의 분포를 알아낼 수 있다면 join strategy 등을 broadcast나 hash로 바꿔버릴 수도 있습니다. (특히 skew가 발생한 상황에서) Spark summit에서의 발표 내용을 보면 최대 100배 빨라진 경우도 있다는데,
  충분히 그럴 잠재력이 있는 major change라는 생각이 듭니다! 이것도 위의 변경사항과 같이 적용되어 특정 query들이 굉장히 빨라졌을 거 같네요.
- [[SPARK-25603] Generalize Nested Column Pruning](https://issues.apache.org/jira/browse/SPARK-25603)
  - Nested column (JSON, map 등) 에 대해서도 column pruning을 적용하고, Parquet에만 적용되던 기존 nested column pruning을 다른 format (ORC 등) 에도 적용하는 ticket 입니다. 요새 점점 비정형 데이터가 늘어나고, JSON format의 데이터가
  더 활발히 돌아다니는 추세인데 이런 게 잘 되면 성능 향상에 큰 이점이 있겠네요!

### Minor features
- [[SPARK-27901] Improve the error messages of SQL parser](https://issues.apache.org/jira/browse/SPARK-27901)
  - 이것은 사실 major에 적어도 될 정도로 중요한 변경 중 하나인데, Spark SQL의 parser인 [ANTLR4](https://www.antlr.org/)가 많은 경우에 굉장히 뜬금없는 에러 메시지를 뱉어주는데 (예를 들어 comma를 빼먹었는데 완전 이상한 곳을 찍어준다던지)
  그걸 개선하는 ticket 입니다. 이 에러 메시지에 낚여서 버린 시간이 얼마인지 생각해보면 이건 정말 중요한 변경인 것 같습니다.
- [[SPARK-27395] New format of EXPLAIN command](https://issues.apache.org/jira/browse/SPARK-27395)
  - 현재 `EXPLAIN` command는 약간 불친절한 면이 있는데 ticket 내용에도 보면 모두가 인지하고 있는 사실인 것 같습니다. EXPLAIN의 readability가 향상되면 더 좋은 코드를 만들어낼 수 있겠죠?
- [[SPARK-25390] Data source V2 API refactoring](https://issues.apache.org/jira/browse/SPARK-25390), [[SPARK-27589] Spark file source V2](https://issues.apache.org/jira/browse/SPARK-27589)
  - 데이터를 불러오는 부분에서도 최적화 및 리팩토링이 이루어진 것 같습니다. Datasource는 주로 DB에서, Filesource는 S3 등의 storage에서 데이터를 가져올 때 사용하는데 어떤 변경점이 있었는지는 정확히 잘 모르겠지만
  매우 기대가 되는 부분 중 하나입니다. 특히 기존 Datasource V1은 좀 오래 전 코드 (1.4 정도) 라서 한 번 갈아엎을 때가 되긴 한 거 같아요.
- [[SPARK-23977] Add commit protocol binding to Hadoop 3.1 PathOutputCommitter mechanism](https://issues.apache.org/jira/browse/SPARK-23977)
  - 예전에는 S3과 연동하면서 성능이 저하되는 경우가 많았는데 (S3N을 쓴다던지, rename 때문에 2번 쓴 다던지) Hadoop 및 Spark, 그리고 AWS에서 아주 가열차게 코드를 수정해주는 덕분에 이제 S3를 써도
  제 성능을 온전히 낼 수 있게 됐습니다. 특히 Hadoop의 zero-rename committer, EMR의 EMRFS committer, 그리고 이번 Spark의 본격적인 S3A connector 지원까지 아주 기대되는 부분이 많은 것 같습니다!


제가 눈여겨보던 Spark 3의 feature 중 하나는 `Arrow` 지원이었는데 (현재는 PySpark에서 JVM 왔다갔다 하면서 SerDe로 발생하는 overhead가 너무 크기 때문) 아직 3.0.0에는 나오지 않은 게 좀 아쉽지만,
워낙 다양하고 많은 최적화가 이루어진 버전이라 Scala 기반의 Spark를 즐겁게 사용하면서 기다릴 수 있을 것 같네요. 또 눈에 띄는 major change가 있으면 요약글로 돌아오도록 하겠습니다!
