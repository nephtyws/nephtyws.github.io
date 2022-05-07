---
title: "AWS ECS에서 force-new-deployment가 제대로 되지 않을 때"
date: 2020-01-29 19:50 +0900
categories: DevOps
toc: true
---

ECS에 배포된 서비스를 재배포하고 싶을 때는 보통 다음과 같은 명령어를 이용합니다.

```shell
aws --profile "${AWS_PROFILE}" ecs update-service --cluster "${CLUSTER}" --service "${SERVICE}" --force-new-deployment --region "${AWS_REGION}"
```

그런데 어느 날 다음 명령어를 이용했는데도 재배포가 되지 않는 현상이 있었습니다. 정확히는 아무 일도 일어나지 않았습니다. 어떤 게 문제인가 싶어 `ECS task definition`을
살펴보니 다음과 같이 설정되어 있는 것을 발견했습니다.
> Minimum healthy percent: 100  
> Maximum percent: 200

즉, task가 요구하는 최소 health가 100이니 재배포가 될 수 없었던 것입니다. 생각해 보면, 재배포가 이루어질 때는 기존 서비스가 내려가고 다시 올라와야 하는데,
그러면 `0 -> 100`이 되어야 합니다. 이를 해결하기 위해서는 `Minimum healthy percent`를 `0`으로 설정해주면 됩니다.

## 결론
- `Minimum healthy percent`를 `0`으로 맞춰보자.
- [참고할만한 Stack Overflow 링크](https://stackoverflow.com/questions/46018883/best-practice-for-updating-aws-ecs-service-tasks)
