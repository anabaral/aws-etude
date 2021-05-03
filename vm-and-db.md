# AWS에서 VM 하나 DB 하나 추가해 연결해 보기

AWS에서 Kubernetes 만들어 쓰거나 VM 하나 만들어 쓰는 데만 익숙하다 보니 기본적인 IaaS 기능에 문외한이 된 느낌.  
여기 그 시행착오를 기록해 봅니다.

## VPC 및 Subnet 생성

이건 이전 작성 문서에서 AWS CLI 및 Cloudformation 문서를 이용해 생성한 그것을 그대로 썼습니다.
[AWS CLI 로 VPC ](./aws-cli.md)

## VM 생성

이건 AWS CLI로 나중에 정리하기고 하고 일단은 웹화면에서 만들어 보았습니다.  
* 가능한 한 싸고 작게 만듭니다.  
* 중요한 것은 bastion 역할을 하는 VM이니 public subnet 에 생성해야 한다는 점입니다.  
* public subnet 에서 생성해야 public ip를 받을 수 있습니다.  
  (이것은 당연히 그렇다기보다 subnet을 public으로 설정할 때 'public ip를 자동으로 부여받기' 체크해서 가능한 것이기도 함)

## DB 생성

Aurora DB 를  serverless 로 만듭니다.  
* 비용절감 옵션을 선택할 수 있던데 저는 5분 넘어가면 중지 모드로 가게 했습니다.  
* 당연히 DB는 private subnet에 생성합니다.
* 저는 VM에서 DB에 붙게 할 겁니다.

## VM과 DB 연결

* mysql client를 설치합니다.  (저는 yum 으로 설치했는데, OS에 따라 apt 만 될 수도 있습니다)
  ```
  [ec2-user@ip-192-168-41-212 ~]$ yum list | egrep '(mysql|mariadb)'
  mariadb.x86_64                         1:5.5.68-1.amzn2               @amzn2-core
  [ec2-user@ip-192-168-41-212 ~]$ yum install mariadb
  ```
* 다음과 같이 연결을 하면 되는데
  ```
  $ mysql -u admin -h ds04226-db-1.cluster-cgxth7zggvw1.ap-northeast-2.rds.amazonaws.com -p
  ```
  안붙네요.
* 이때부터 뻘짓을 하기 시작합니다.
  - VM이 속한 public subnet의 라우팅 테이블을 체크.
    + 192.168.0.0/16 이 local 로 되어 있는데 이건 맞음. 이 local은 VPC 범위라고 생각하면 됨.
    + 그 외엔 igw 로 연결. 이것도 맞음.
    + 서브넷 연결 체크. public subnet에 명시적 연결하는 걸로 등록되어 있음.
      어차피 나머지 private subnet들은 기본 라우팅 테이블을 통해 연결되어 있음.
    + 정상.
  - DB가 속할 private subnet의 라우팅 테이블을 체크.
    + 192.168.0.0/16 이 local. OK.
    + 그 외엔 nat-gw 로 연결. OK.
    + 명시적 서브넷 연결은 없음.
    + 정상.
  - 보안그룹을 확인.
    + VM 을 생성할 때 만들어 둔 보안그룹이 있음. launch-wizard-23
    + 이것이 DB의 보안그룹으로도 등록되어 있음. 만들 때 그렇게 했던 기억이..
    + 그 보안그룹의 인바운드 규칙을 보자.
      . SSH 22 접근에 대해 0.0.0.0/16 --> 이건 내 PC쪽에서만 붙게 일단 고쳐놓자.
      . 추가를 하자. MYSQL/AUrora 3306 접근에 대해 소스를.. 같은 보안그룹으로.
    + 여기까지 하니 접근 된다!
    + 저는 지금까지 같은 보안그룹에 있는 것으로 충분하다고 생각했었는데, 실은 인바운드 규칙 등이 도와줘야 하는 거였네요.
* 다시 연결 시도했고, 시간이 10초 이상 걸려 안되나 싶었지만 접속이 되었습니다.
