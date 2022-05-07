---
title: "Airflow 오류 - The CSRF session token is missing"
date: 2020-01-29 18:30 +0900
categories: Data
toc: true
---

어느 날 Airflow에 접근했더니 갑자기 다음과 같은 오류 메시지가 나오기 시작했습니다.

> Bad request - The CSRF session token is missing.

CSRF 설정을 건드린 적이 없는데 갑자기 안 된다고 나오니 조금 당황스러웠습니다. 처음엔 브라우저 문제인 줄 알고 브라우저를 마구 괴롭혔는데
(전체 재설정, private mode에서 시도, 브라우저 바꾸기 등) 아무리 해도 정상으로 돌아오지 않아서 브라우저 문제는 아니라는 걸 알게 되었습니다.  

이후 Airflow 서비스 전체를 재배포했는데 그래도 여전히 해결되지 않았습니다. 조금 더 찾아보니, 해당 CSRF 설정은 Airflow가 웹 서버로 사용하는
Flask에서 제공하는 기능이라는 것을 알 수 있었습니다. 그래서, Airflow 설정 중 cookie 설정과 관련된 부분을 다음과 같이 바꿔주었습니다.

```ini
# Set secure flag on session cookie
cookie_secure = True to False

# Set samesite policy on session cookie
cookie_samesite = Lax to None
```

저는 원래 해당 설정을 `True`, `Lax` 로 사용 중이었는데, 이 부분을 `False`, `None` 으로 변경하고 Airflow 서비스를 재배포하였습니다.
이내 정상으로 돌아온 Airflow를 확인할 수 있었습니다. 그리고 다시 설정을 원래대로 복구했는데 (`True`, `Lax` 로) 아무 문제 없이 잘 되는 것을
확인하였습니다!

## 결론
- `cookie_secure`, `cookie_samesite` 설정을 좀 더 약하게 조절하고 서비스를 재배포해보자.
