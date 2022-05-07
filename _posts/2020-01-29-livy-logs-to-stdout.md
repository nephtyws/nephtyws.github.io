---
title: "Apache Livy에서 Spark job stdout log를 보는 법"
date: 2020-01-29 19:30 +0900
categories: Data
toc: true
---

저는 Spark cluster에 job을 제출할 때 [Apache Livy](https://livy.incubator.apache.org/)라는 프레임워크를 사용합니다. Livy는 RESTful API를 이용하여
Spark cluster에 job을 제출할 수 있게 하는 도구입니다. 다만 `spark-submit`을 이용하여 작업을 제출할 때와 약간의 차이점도 있는데, 일단 Spark resource manager에서
`owner`가 `livy`로 나온다던가, log를 보려면 Livy web UI를 이용하거나 API를 이용하여 받아와야만 하는 등의 차이가 있습니다.  

Spark job에서 필요에 의해 작업 진행 상황을 봐야하는 경우 보통 `println` 등의 기본 함수를 써서 `stdout`으로 출력하는데요. 이 경우 `spark-submit`로
제출했을 때는 문제가 없지만 `Livy`로 제출하는 경우에는 기본 함수를 이용해서 출력한 값은 `stdout` log에서 확인이 되지 않습니다.  

log를 제대로 출력하려면 각 언어에서 제공하는 `logger class`를 사용하면 되는데요. Scala의 경우에는 `log4j`를 이용하면 됩니다.

## 결론
- `log4j` 와 같은 logger class를 이용하면 Livy로 job을 제출했을 때도 정상적으로 stdout log를 확인할 수 있다.
- [참고할만한 Stack Overflow 링크](https://stackoverflow.com/questions/59650722/read-spark-stdout-from-driverlogurl-through-livy-batch-api/)
