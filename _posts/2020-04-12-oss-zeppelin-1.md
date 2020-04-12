---
title: "[ZEPPELIN-4611] Fetching rows with newline character (\n) breaks entire table"
date: 2020-04-12 16:50 +0900
categories: OSS
---

얼마 전 사내에서 데이터 분석가분의 troubleshooting을 도와주다가 Zeppelin의 버그를 발견했습니다. 바로 table content에 개행문자 (\n) 가 있으면 전체 table이 깨져 보이는 버그였습니다. 워낙 원인이 명확해보이는 버그라 망설임없이 Zeppelin JIRA에 [report](https://issues.apache.org/jira/browse/ZEPPELIN-4611)를 올렸고, maintainer 분이 금방 패치해주셨습니다. 사실 너무 고치기 쉬워보이는 내용이라 직접 PR을 작성할 생각도 있었는데, maintainer 분이 굉장히 빠르게 PR을 올려주시더군요...

일단 실제 사용 환경에서는 `translate` 함수를 이용해 각종 개행문자를 whitespace로 치환해주니 멀쩡히 동작하는 것을 확인했습니다. 근데 이런 문제가 Zeppelin 0.8.2까지 해결되지 않았다는 게 여러모로 신기하네요. 굉장히 후진 console을 제공하는 Athena도 (?) 이미 잘 대응되어 있던 문제인데요.

OSS에 contribution 할 아이디어는 굉장히 많은데 아직 하나도 실현된 게 없습니다. 이 포스트를 기점으로 OSS에 기여하겠다는 의지를 불태울 계획입니다. 다음엔 Spark에 contribution한 내용으로 찾아뵙겠습니다!
