# AWS CLI 에서 이것저것


## AWS CLI 설치

여기를 참조해서 진행: https://aws.amazon.com/ko/cli/

64비트 실행 프로그램을 다운받아 설치.

설치 후 인증/권한 위한 명령 실행

```
C:\Users\rindon>aws configure
AWS Access Key ID [None]: AKIAYUPXW4SKDIE9D01X                        <-- 적당히 바꾼 것이니 염려 마세요
AWS Secret Access Key [None]: gA+xBBs1XxxxYyyyyzzzAaaaa2ooo0pppppEaan <-- 적당히 바꾼 것이니 염려 마세요
Default region name [None]: ap-northeast-2                            <-- 서울
Default output format [None]: 그냥엔터                                 <-- json/yaml/text/table 선택가능
```

이 입력 결과는 홈디렉터리 밑의 ```.aws``` 밑에 파일로 저장됩니다

아마 지겹게 쓰게 될 명령어:
```
> aws eks help
```

### AWS EKS 클러스터 셋업

다음을 참조하는데 100% 따라하지는 않게 될 것 같음:
https://docs.aws.amazon.com/ko_kr/eks/latest/userguide/getting-started-console.html
- 위 링크는 cloudformation 으로 VPC 나 subnet들을 생성하고 인터넷 연결 등을 설정하는데 기본적으로 갖출 것을 갖추고 있음.
- ```
  aws cloudformation create-stack \
  --stack-name ds04226-eks-vpc-stack \
  --template-url https://s3.us-west-2.amazonaws.com/amazon-eks/cloudformation/2020-10-29/amazon-eks-vpc-private-subnets.yaml
  {
      "StackId": "arn:aws:cloudformation:ap-northeast-2:592806604814:stack/ds04226-eks-vpc-stack/c3521c70-a1ab-11eb-9521-0a386e742970"
  }
  ```
  이렇게 한 것만으로 VPC 를 포함한 리소스들이 만들어집니다.
  * VPC 1개
  * Subnet 4개 (AWS::EC2::Subnet)
    - public subnet 2개
    - private subnet 2개
  * 라우팅 테이블 4개 (AWS::EC2::RouteTable)
    - 기본 1개 (그냥 자동으로 존재하는..? cloudformation 에서는 기술되지 않음)
    - Public 1개가 2개의 public subnet에 연결
    - Private 2개가 각기 하나씩 private subnet에 연결
    - (cloudformation 상에선 연결 도 하나의 개체 -- AWS::EC2::SubnetRouteTableAssociation -- 로 취급해서 4개의 연결이 선언되어 있음)
    - outbound 통신을 위해 Gateway 들과 연결 (이 연결도 개체 -- AWS::EC2::Route )
      * public 라우팅테이블은 인터넷 게이트웨이와 연결
      * private 라우팅테이블들은 하나씩 NATGateway와 연결
  * 인터넷 게이트웨이 1개 (AWS::EC2::InternetGateway)
    - public 라우팅테이블과 연결
    - cloudformation 상에선 AWS::EC2::VPCGatewayAttachment 개체와 연결되어 있음. 이 개체가 다른 개체와의 연결을 구성.
  * EIP 2개 (AWS::EC2::EIP)
    - 각기 하나씩 NATGateway에 할당
  * NATGateway 2개 (AWS::EC2::NatGateway)
    - 각기 하나씩 public subnet 안에 존재하면서
    - 각기 하나씩 private subnet 과 연결되어 있음 (라우팅 테이블 / 라우트 로) 
  * Network ACL 1개
    - 4개의 subnet에 연결
  * 보안 그룹 1개 (AWS::EC2::SecurityGroup)

