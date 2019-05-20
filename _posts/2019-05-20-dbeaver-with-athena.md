---
title: "AWS Athena와 잘 어울리는 DB 클라이언트 - DBeaver"
date: 2019-05-20 23:03 +0900
categories: Database
---

### AWS Athena와 잘 어울리는 DB 클라이언트 - DBeaver

최근에 회사 데이터 플랫폼 (저희 팀이 관리하고 있는..) 을 On-premise Hadoop 환경에서 Managed AWS (EMR) 환경으로 모조리 이전했습니다. 
예전에는 Azure 위에서 VM을 계속 띄워놓고 그 위에 Cloudera Hadoop을 설치하는 형태로 사용했었는데요. 데이터 분석가분들께서 플랫폼을 사용하실 때 Cloudera 배포판에 같이 딸려오는 Hue와 Impala를 이용해 데이터 플레이그라운드 (?) 를 제공해드렸었는데, AWS EMR로 넘어오면서부터는 필요할 때만 클러스터를 올려 쓰는 형태로 변경되었기 때문에 더 이상 Hue와 같은 대시보드를 계속해서 제공할 수 없게 되었습니다.  

이에 대한 대안으로 [AWS Athena](https://aws.amazon.com/ko/athena/)를 도입하여 HDFS (사실은 S3) 에 있는 테이블들을 이전과 같이 SQL-like 인터페이스를 이용하여 조회할 수 있도록 제공하기로 했습니다. 그러나 항상 쿼리를 날리기 위해 AWS console에 접속할 수는 없는 노릇이지요! 좋은 도구에는 좋은 클라이언트가 필수인 법입니다. 어떤 클라이언트를 쓰면 좋을까 고민하다가 회사의 분석가분께서 추천해주신 클라이언트가 바로 [DBeaver](https://dbeaver.io/) 입니다.  

사실 저는 Jetbrains 사의 제품을 좋아해서 DataGrip을 이용할까 생각했었는데, 확인해보니 DataGrip에는 Athena 인터페이스가 없더군요. (혹은 있는데 제가 못봤을 수도 있습니다.) DBeaver는 아주 직관적인 인터페이스 (DB 클라이언트라고 하면 딱 떠올릴 수 있는 그런 UI) 와 함께 무난한 성능, 그리고 Java 기반이라 모든 OS에서 다 사용할 수 있다는 장점이 있습니다.  

다만 Athena의 모든 기능을 지원하지는 않는다는 문제점이 있는데요. 대표적으로 [Workgroup](https://docs.aws.amazon.com/athena/latest/ug/user-created-workgroups.html)을 사용할 수 없는 문제가 있습니다. Athena는 쿼리가 읽어들인 데이터 1TB당 5$로 과금이 꽤나 빡빡한 편인데요. 이를 잘 관리하기 위해서는 사용자 그룹별로 Workgroup을 나눠 과금을 모니터링할 필요가 있습니다. 그러나 DBeaver가 사용하는 Athena JDBC는 최근에 추가된 Workgroup 기능을 지원하지 않아 기본 그룹인 `primary` 그룹으로만 쿼리가 되는 문제가 있었습니다...만,  

제가 방금 그 문제를 고쳤기 때문에 최신 버전의 DBeaver에서는 workgroup을 정상적으로 사용하실 수 있게 됐습니다. (참고: [Update plugin.xml to download latest version of Athena JDBC driver to support change workgroup](https://github.com/dbeaver/dbeaver/pull/5945))  

완전 날로 먹은 PR이지만 어쨌든 최신 버전에서는 정상적으로 돌아갈 걸 생각하니 기쁘네요.  

Athena와 어울리는 DB 클라이언트를 찾고 계셨다면 이번 기회에 DBeaver를 사용해보시는 건 어떨까요?
