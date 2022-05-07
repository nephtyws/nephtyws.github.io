---
title: "Airflow의 execution_date에 대하여"
date: 2020-04-12 17:20 +0900
categories: Data
toc: true
---

[Airflow](https://airflow.apache.org/)는 Airbnb에서 시작된 Job orchestration framework로 데이터 엔지니어링 사이드에서 꽤나 많이 사용하는 도구 중 하나입니다. 저도 현업에서 production용으로 이미 사용하고 있고, 20+ DAGs, 200+ tasks를 매일매일 돌리고 있습니다. 이와 비슷한 도구로는 Spotify에서 만든 [Luigi](https://github.com/spotify/luigi)가 있습니다. 슈퍼 마리오에서 배관을 타고 들어가는 Luigi 캐릭터를 보고 지은 이름인 듯 합니다.

어쨌든 두 가지 모두를 현업에서 운영해보니 각각의 도구가 가진 장단점이 명확하게 보이기 시작했습니다. 먼저 Airflow가 더 좋은 점은 Airflow에는 scheduler가 integration 되어 있어 외부 triggering에 의존하지 않고 직접 각 job을 tracking 할 수 있다는 것이 가장 명확한 장점입니다. 보통 Luigi를 사용할 때는 Luigi 자체는 job orchestrator로 사용하고, 이러한 Luigi task를 Jenkins같은 도구를 이용해서 trigger 하곤 합니다. 이렇게 사용했을 때의 문제는 Luigi와 Jenkins 사이의 결합이 탄탄하지 않고, (여기선 coupling이 tight해야 좋습니다!) job triggering이 매끄럽지 않으며, Jenkins가 SPoF가 된다는 사실입니다. 보통 Jenkins 안에는 다른 job도 섞여있기 마련이니, 그게 싫으면 별도의 Luigi용 Jenkins를 두어야 한다는 거겠죠.

하지만 그 점 빼고는 모든 면에서 Luigi가 낫다는 느낌을 받기도 했습니다. 일단 task간의 dependency check가 용이하고, 굳이 DAG 개념에 얽매이지 않고 task끼리 inter-dependency를 용이하게 걸어줄 수 있으며, 오늘 언급할 주제인 execution_date, 즉 marker를 check하는 것도 아주 자유도가 높아서 프로그래머가 원하는대로 로직을 설계할 수 있게 되어 있습니다.

제가 어떤 근거로 이러한 이야기를 하는 지에 대해 이제 천천히 얘기해보겠습니다.

## Airflow execution_date와 marker
대부분의 Job orchestration framework는 `marker` 라는 것을 이용해서 특정 job의 success/failure 여부를 확인하게 됩니다. Spark나 Hive job을 이용하다보면 HDFS에 `_SUCCESS` 와 같은 dummy file 혹은 folder가 만들어지는 것을 심심찮게 확인할 수 있는데, 이것은 Spark/Hive가 내부적으로 HDFS에 write 하는 작업이 성공했는지 실패했는지 여부를 모든 node에서 명시적으로 확인할 수 있게 하기 위하여 `marker`를 HDFS에 남겨놓은 것입니다.

유사한 방식으로, `Luigi`나 `Airflow`와 같은 framework들도 marker를 이용해서 job의 상태를 확인할 수 있습니다. Luigi에서는 아예 `file marker`라는 개념을 지원해서 위처럼 특정 경로에 어떤 파일의 존재 유무로 job의 상태를 확인할 수 있기도 하지만, 이것보다 좀 더 현대적인 방법은 바로 `DB`를 이용하는 것입니다. 특히 요즘처럼 하나의 HDFS를 이용하지 않고 S3와 같은 distributed object storage로 이용할 때는 더더욱 DB를 이용하는 것이 신뢰성이 높은 방법입니다. 왜냐하면 S3와 같은 object storage들의 consistency model은 [eventual consistency](https://en.wikipedia.org/wiki/Eventual_consistency)이기 때문에 자칫하면 큰 참사가 날 수도 있습니다.

`Luigi`와 `Airflow` 모두 DB 기반의 marker를 제공하긴 하지만, 이 두 개의 도구 사이에는 아주 큰 차이가 있습니다. 먼저, Luigi는 custom marker를 지원합니다. 사실 custom marker를 지원한다기보단 framework 차원에서 정해진 marker rule이 잘 없습니다. 사용자가 직접 marker DB를 지정하고, marker content를 지정해서 사용자의 job에서 marker를 조회해서 dependency를 확인하는 느낌입니다.

반면에, `Airflow`는 `exeuction_date`라는 one and only marker를 기본적으로 제공합니다. 기본적으로 Airflow는 `PostgreSQL`을 marker DB로 사용하고, 각 task instance가 success/failure 했는지 여부를 marker DB에 기록합니다. 이는 Airflow의 Web UI에서 간편하게 확인할 수 있죠.

여기까진 좋은데, Airflow의 execution_date는 `UTC based datetime`입니다. 이게 왜 문제가 될까요?

## Airflow에서 DAG 혹은 task 간의 dependency check
Airflow에서 execution_date는 단순히 해당 TI (task instance) 의 시작일을 알려주는 것이 아닙니다. Airflow는 execution_date를 통해 각 TI에 id를 부여할 뿐만 아니라, 이것을 통해 TI를 unique하게 구분할 수 있습니다. (e.g. `TestJob-20200412170000`) 그럼 다음과 같은 상황을 예로 한 번 들어봅시다.

> 매일 02시에 한 번 동작하는 batch job이 있다. 그리고 매일 08시에 동작하는 batch job은 이전 작업 (02시 batch job) 이 반드시 성공해야만 동작할 수 있다. 어떻게 확인할 수 있을까?

Luigi였다면 그냥 `TestJob-20200402` 처럼 `yyyymmdd` 형태의 marker를 조회하면 쉽게 dependency check를 할 수 있었을 겁니다. 혹은 이게 못미덥다면, `yyyymmddHHMMSS` 단위로 해당 job의 marker를 조회하면 됩니다. 그런데, Airflow에서 dependency check를 하려면 다음과 같이 해야합니다.

```python
    dag_time: Time = dag['schedule_time']
    dag_timedelta = timedelta(hours=dag_time.hours, minutes=dag_time.minutes)
    my_time = timedelta(hours=watcher_time.hours, minutes=watcher_time.minutes)

    yield ExternalTaskSensor(
        task_id=f"waiting_{dag['name']}",
        external_dag_id=dag['name'],
        execution_delta=my_time - dag_timedelta,
        timeout=600,
    )
```

즉, 내가 check하고 싶은 DAG의 time을 가져와서 그것의 delta를 직접 계산한 후에 Sensor에 넣어줘야 합니다. 물론 이렇게 하면 대부분의 경우를 cover 할 수 있긴 한데, execution_date의 개념을 제대로 알지 못하면 헷갈릴 수도 있습니다.

또한 이 방법은 schedule time을 기반으로 하기 때문에, manual trigger job에 대해서는 watch 할 수 없다는 것이 문제입니다. 이때문에 job이 실패한 경우 실패한 TI에서 모든 걸 해결해야만 하고, 그냥 처음부터 다시 돌리고 싶어서 clear를 하지 않고 manual execution을 해버리면 dependency job을 제대로 찾지 못해서 job이 fail 합니다.

이쯤되면 `ExternalTaskSensor`를 잘못 만들었다는 생각이 솔솔 들고 있습니다. 위의 예제는 DAG를 watch하는 예제인데, 이걸 task까지 확장시키면 코드도 엄청 늘어나고 DAG 안의 task도 엄청 늘어나서 보기가 썩 좋진 않습니다.

## 그래도 Airflow는 잘 만든 도구다
갑자기 급 마무리하는 느낌이 들긴 하지만, 어쨌든 Airflow는 production 환경에서 사용하기 적합한 도구입니다. 그런데 다른 orchestration framework에 익숙하신 분이라면 Airflow를 사용하실 때 조금 주의하실 필요는 있겠습니다. 우리가 기존에 사용하던 framework와 약간 다른 면이 있기 때문입니다.

### 결론
> Airflow의 execution_date는 UTC based datetime이고, TI의 unique ID이며, 실행 시마다 정확한 execution time을 알아야 DAG/task를 watch 할 수 있다.

제가 인지하고 있는 다양한 불편함을 언젠가는 직접 PR을 날려서 해소할 그날이 왔으면 좋겠네요. 모쪼록 이 글이 execution_date를 이해하시는 데 도움이 되었으면 좋겠습니다!
