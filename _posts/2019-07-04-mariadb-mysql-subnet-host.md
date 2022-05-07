---
title: "MySQL/MariaDB에서 유저에게 multiple host를 부여하는 방법"
date: 2019-07-04 21:15 +0900
categories: Database
toc: true
---

### 1. Subnet mask 이용하기
다음과 같이 설정하면 여러 대역대에 대해서 접근을 허용할 수 있습니다. 그러나 `mysqluserclone` 같은 오래된 유틸리티에서는 이러한 방식을 지원하지 않으니 주의하세요.
> e.g.) 'username'@'10.12.0.0/255.255.0.0', 'username'@'192.168.1.0/255.255.255.0', 'username'@'52.0.0.0/255.0.0.0'

### 2. '%' 이용하기
% wildcard를 통해서도 여러 대역대에 대해 접근 허용이 가능합니다. 단, 사용 방식에 주의하세요. 아래에 예제가 나와있습니다.
> e.g.) 'username'@'10.12.0.%', 'username'@'10.12.%', 'username'@'10.%'

### 3. CIDR 이용하기
**CIDR을 이용하시면 안 됩니다.** MySQL/MariaDB에서 지원하지 않습니다. (꼭 지원했으면 좋겠네요..!)
> e.g.) 'username'@'10.12.0.0/16' (지원 X)
