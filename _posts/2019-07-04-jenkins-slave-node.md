---
title: "Jenkins slave node 구성하기 - Troubleshooting 포함"
date: 2019-07-04 20:37 +0900
categories: DevOps
---

최근에 회사에서 [Jenkins](https://jenkins.io/)를 활발히 쓸 기회가 생겼습니다. 목적은 AWS에 배포된 EC2를 Slave node로 등록해서 배포를 자동화하려는 것이었는데요. 의외로 한 번에 설정하기가 쉽지 않더라고요. 예상치 못한 문제도 발생했고요.  

그래서, 지금까지의 삽질 경험을 기록으로 남기고, 제 자신을 비롯한 후대에게 지식을 공유하기 위해서 글을 쓰게 되었습니다.

### Jenkins slave node 구성, 다음 단계만 따라하세요
- 준비물 : Master node, Slave node
- Master node에 직접 접속해서 (SSH) jenkins user의 private key를 가져옵니다. (`/home/jenkins/.ssh/id_rsa`)
  - 아직 없다면 하나 만들어주세요. (`ssh-keygen`)
- Master node의 public key도 가져옵시다. (`/home/jenkins/.ssh/id_rsa.pub`) 곧 사용할 예정이니 보관해두세요.
- 이렇게 얻은 Master jenkins user의 private key를 Jenkins console -> crendentials에 등록해주세요. (`Username with SSH key`)
- 그 다음, Slave node에 직접 접속해서 jenkins user를 생성해줍니다. (`useradd jenkins`)
- 그리고, Slave jenkins user의 authorized_keys에 Master node의 public key를 등록해주세요. (`/home/jenkins/.ssh/authorized_keys`)
  - 파일이 없으면 새로 만들어줍시다.
- 이때, `.ssh` 폴더와 그 내용물의 권한을 꼭 확인하세요! 권한이 이것보다 크면 Master -> Slave node 접속이 실패할 수도 있습니다.
  - `644` - id_rsa.pub, known_hosts
  - `700` - .ssh 폴더
  - `600` - id_rsa, authorized_keys
- 마지막으로, Master jenkins user의 known_hosts에 Slave host를 등록해줍시다.
  - 이를 위한 가장 쉬운 방법은 `ssh jenkins@slavehost` 입니다. `yes`를 입력하면 자동으로 등록돼요.
  - 만약 동일한 host나 IP를 사용하고 있어서 이전에 known_hosts에 등록된 적이 있다면, `ssh-keygen -R host` 처럼 갱신해주면 됩니다.
- 이제 Jenkins console에서 등록하시면 됩니다!

굳이 사진과 함께 친절한 설명을 하지 않은 이유는, 다른 블로그에 그런 내용이 아주 많아서입니다. 그리고 Slave node를 여러 대 만들어야할 운명에 처하셨다면, 위의 과정을 [Packer](https://www.packer.io/)로 구성해놓으시면 매우 편해요! 저희 회사에서는 실제로 Slave node용 AMI를 만들어 사용하고 있습니다. 그러면 유저 등록, Java 설치 등 귀찮은 일을 좀 덜 수 있게 됩니다.

### 실패하셨다고요?
위에 이미 모든 단계 (와 제가 겪은 삽질을 방지하기 위한 여러 방법들) 를 적어놓긴 했는데, 혹시 Master -> Slave 접속이 실패해서 slave node가 offline으로 뜬다면 다음과 같은 것들을 확인해보세요. (특히 AWS 환경이라면!)

- Slave node security group의 outbound 설정을 확인해보세요. Master도 마찬가지입니다.
- Master와 Slave가 서로 통신할 수 있는 상태인지 확인해보세요. 특히, SSH port가 열려있는지 확인해보세요.
- `Server rejected the 1 private key(s) for jenkins` 라거나, `Public key - permission denied` 같은 에러는 진짜 권한이 부족하거나 private key가 달라서 생기는 에러일수도 있지만, 경험상 특정 설정이 이상해서 뜨는 메시지였습니다.
  - 특히 `.ssh` 폴더와 내용물의 권한, Jenkins slave node 설정에서 credential 설정 등...

그럼 Jenkins와 함께 행복한 `CI` 되시길 기원하겠습니다!
