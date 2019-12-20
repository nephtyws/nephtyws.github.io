---
title: "Airflow에서 superuser 만드는 법"
date: 2019-12-20 19:30 +0900
categories: Data
---

Airflow에서 다음과 같은 설정값을 통해 간단한 계정 기반 인증 시스템을 구현할 수 있습니다.

```editorconfig
# Set to true to turn on authentication:
# https://airflow.apache.org/security.html#web-authentication
authenticate = True
auth_backend = airflow.contrib.auth.backends.password_auth
```

문제는 인증 시스템을 이용하면 Airflow를 초기화하는 단계에서 계정을 따로 만들어주어야 한다는 것입니다. 그렇지 않으면
Airflow 웹 서버에 접근할 수 없습니다. 이런 작업을 CLI에서 해줘도 되지만, 저는 Docker 기반으로 Airflow를 사용하고
있어서 매번 CLI를 호출하고 싶지 않았습니다. 그래서 다음과 같이 계정을 생성해주는 간단한 Python 코드를 작성하여 계정을
만들어주었습니다.

```python
# Create Airflow user
user = PasswordUser(models.User())
user.username = "airflow"
user.email = EMAIL
user.password = PASSWORD
user.superuser = True

session = settings.Session()
session.add(user)
session.commit()
session.close()
```

특히, `superuser = True` 를 하지 않으면 관리자 권한이 없는 일반 계정이 만들어져서 웹 서버에서 Airflow를 관리하는 데 있어
상당한 제약이 있습니다. 관리자 계정을 만들고 싶으시면 꼭 저 값을 `True`로 변경해주세요. 기본값은 False 입니다.

#### 결론
- `superuser = True`를 통해 관리자 계정을 생성할 수 있다.
- `airflow.cfg`에서 `authenticate = False`로 설정해주었다면, 기본적으로 모두가 관리자 권한을 갖게 된다.
