---
title: "Airflow 오류 - Some workers seem to have died and gunicorn did not restart them as expected"
date: 2019-12-20 19:05 +0900
categories: Data
toc: true
---

ECS에 Airflow를 설정하던 도중 다음과 같은 오류 메시지를 만났습니다.
> Some workers seem to have died and gunicorn did not restart them as expected

로그를 살펴보니, Airflow의 웹 인터페이스를 담당하는 `gunicorn` worker들이 정상적으로 실행되지 않았고, 이로 인해 웹 서버 자체가 사망하는 일이 발생했습니다.
설정이나 ECS task definition을 봐도 딱히 잘못된 부분이 없어보여서 조금 헤맸는데, 약간
살펴본 결과 **메모리가 부족해서 생기는 문제였습니다.** 웹 서버 container에 350MiB를 할당하고, gunicorn worker 수는
기본 설정값대로 4개를 사용 중이었는데 혹시나 해서 worker 개수를 3으로 줄이니 정상 실행되는 것을 확인할 수 있었습니다.

참고로, gunicorn worker 수는 `airflow.cfg`의 다음 섹션에서 찾으실 수 있습니다.

```ini
[webserver]
# Number of workers to run the Gunicorn web server
workers = n
```

## 결론
- 메모리를 좀 더 할당하거나 worker 개수를 줄이면 해결할 수 있다.
- 참고로 600MiB / workers = 3 으로 설정해서 잘 쓰고 있습니다. (`docker stats`로 확인해본 결과 약 350MiB 정도
점유하고 있는 것을 확인하였습니다)
