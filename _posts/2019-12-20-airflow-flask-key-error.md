---
title: "Airflow 오류 - The session is unavailable because no secret key was set"
date: 2019-12-20 19:15 +0900
categories: Data
toc: true
---

ECS에 Airflow를 설정하던 도중 다음과 같은 오류 메시지를 만났습니다.
> RuntimeError: The session is unavailable because no secret key was set. Set the secret_key on the application to something unique and secret.

Airflow의 웹 서버인 Flask가 사용할 session secret key가 제대로 설정되지 않아서 생기는 오류 메시지라고 생각했습니다.
저같은 경우 secret key와 같은 crendential 정보는 외부 (Git 저장소 등) 로 유출되면 안 된다고 생각해서 `airflow.cfg`에
설정하지 않고 ECS task definition에서 [AWS SSM Parameter Store](https://docs.aws.amazon.com/ko_kr/systems-manager/latest/userguide/systems-manager-parameter-store.html)
에 저장되어 있는 key를 불러온 후 entrypoint에서 가져다 사용하는 방식으로 설정했습니다. 코드로 표현하면 다음과 같습니다.

> export AIRFLOW__WEBSERVER__SECRET_KEY="${FLASK_SECRET_KEY}"

분명히 key가 설정되어 있음에도 불구하고 계속해서 오류를 뱉는 것을 보고 key 자체가 잘못된 값이 아닌가하는 의심을 하기
시작했습니다. 저같은 경우 key를 따로 만들기 귀찮아서 Fernet key와 Flask session secret key를 동일한 값으로 사용하고 있었는데,
아무래도 Fernet key가 너무 길기도 하고 특수문자를 포함하고 있기도 해서 이 부분이 문제일 거라고 의심하였습니다. 그래서
Flask session secret key가 보통 어떻게 만들어지는지 검색을 통해 찾아봤습니다. 다음과 같이 만들어지더군요.

```python
import secrets  
secrets.token_urlsafe(16)
# 'Drmhze6EPcv0fN_81Bj-nA'
```  

Fernet key는 44 byte 정도 되는데, 통상적으로 사용하는 Flask session secret key의 길이는 16-24 byte 정도 되는 것을
확인할 수 있었습니다. 위처럼 secret key를 만들어주고 그 값을 사용하였더니 문제가 바로 해결되었습니다.
 
## 결론
- Flask session secret key의 값이 너무 길거나, urlsafe 하지 않으면 값이 제대로 설정되어 있어도 Flask에서 key를 제대로
불러오지 못할 수도 있다.
- `secrets.token_urlsafe` 함수를 이용해 key를 만들어 쓰면 좋다. (단, Python 3.6 이상부터 지원) 그 이하 버전에서는 다음과 같이
key를 만들어도 된다.
```python
import os
os.urandom(12).hex()
# 'f3cfe9ed8fae309f02079dbf'
```
