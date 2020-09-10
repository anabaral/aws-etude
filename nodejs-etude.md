# nodejs 와 mysql 등등을 설치해서 개발 연습 해 보기

# 내 운동장 만들기

```
$ kubectl create ns selee
```
이렇게 하면 나만의 격리된 공간에서 테스트를 할 수 있음.  
load balancer라도 설치하면 별도 비용이 부과되겠지만 
일단은 컨테이너에 직접 들어가 ```curl``` 로 테스트한다고 치고..

아래는 선택 사항. 기본 네임스페이스를 여기로 두자는..
```
$ kubectl config set-context --current --namespace=selee
```

# nodejs 설치
```
$ helm repo list  # 헬름 저장소 확인
$ helm repo add bitnami https://charts.bitnami.com/bitnami  # 저장소가 없으면 등록
$ helm repo add stable  https://kubernetes-charts.storage.googleapis.com/   # 저장소가 없으면 등록

$ helm search repo node    # 'node' 라는 키워드로 뭐 깔게 있나 확인 (등록된 저장소에서 검색)
```

최신을 원하면 ```bitnami/node```를 까는게 낫네. 검색하고 좀 헤메다 보니 https://bitnami.com/stack/nodejs/helm 도착.
상세한 설명을 원하니 여기까지 가야 함: https://github.com/bitnami/charts/tree/master/bitnami/node/#installing-the-chart

잘 뜯어보면 선택해야 할 것이 몇 가지 있음.
- persistent volume 을 연결할 것인가 (기본은 불필요)
- mongodb 차트를 설치할 것인가 (기본은 설치)
  * 자동으로 8Gi 짜리 PVC를 생성하는데 이 사이즈를 어떻게 할까
- 내가 만든 앱을 어떻게 깔 것인가
  * 그냥 설치하면 initContainer 가 git 클라이언트 역할을 해서 github에서 샘플 앱을 받아 ```/app``` 디렉터리에 넣음.
  * 이걸 그대로 실행하는 것임. 역시 신박하다.. 스크립트 언어의 위용.. 우리도 이렇게 하면 될 것 같다.
  * 최종적으로는 우리도 gitea에 우리의 소스를 가져다 놓고 등록해서 쓸 수 있겠네.

일단은(!) 아무 선택도 하지 말고 그냥 깔아보자.
```
$ helm install mynode bitnami/node         # 혹시 기본 네임스페이스 설정 안했으면 -n <namespace> 옵션 추가. 
coalesce.go:160: warning: skipped value for tolerations: Not a table.
NAME: mynode
LAST DEPLOYED: Thu Sep 10 06:57:52 2020
NAMESPACE: selee
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
1. Get the URL of your Node app  by running:

  kubectl port-forward --namespace selee svc/mynode 80:80
  echo "Node app URL: http://127.0.0.1:80/"
```

근데 이 샘플 앱 엄청 큰 녀석인데... https://github.com/bitnami/sample-mean

## PC 에서 k8s 붙을 수 있게 셋업

뜬금없지만 필요가 생겨서 넣음. Load Balancer 를 쓰지 않으면서 테스트를 손쉽게 하려면 
위에 설치 메시지에 언급된 ```kubectl port-forward``` 를 활용해야 하는데
이걸 제대로 쓰고 싶으면 본인 PC에서 연결하는 게 제일 낫겠다.

먼저 AWS CLI 와 kubectl 을 설치한다. 이건 검색하면 손쉽게 설치 가능하니까 설명 생략.

그 다음에 해야 하는게 AWS 권한 획득.
```
$ aws configure
AWS Access Key ID [None]: AKIAIOS______MPLE
AWS Secret Access Key [None]: wJalrXUtn______________YEXAMPLEKEY
Default region name [None]: ap-northeast-2
Default output format [None]: json
```
여기 입력하는 값들은 보안 키를 생성해서 어쩌구 저쩌구 했던 거 같은데 
나는 이미 배스천 서버에 이 설정이 되어 있으니 ( $HOME/.aws 밑의 파일들 ) 거기 내용을 참조하면 됨.

그 다음이 kubectl 권한 획득
```
> aws eks get-token --cluster-name mta-cluster   # 이게 꼭 필요한 실행인지 확실하지 않음. 먼저 수행했었기에 적어둠.
> aws eks update-kubeconfig --name mta-cluster

# 시험 실행
> kubectl get nodes
```

이제 위에서 하라고 하는 걸 해 보자.
```
> kubectl port-forward --namespace selee svc/mynode 8080:80    # 80은 좀 심했고, 8080으로 해 보자.
```

이렇게 하면 http://localhost:8080 접근으로 아래처럼 화면이 뜬다.

![샘플앱_초기화면](https://github.com/anabaral/aws-etude/blob/master/img/20200910_nodejs_sample_app_screen.png?raw=true)

이제 남은 건 몇 가지 바꿔가면서 테스트해 보는 것.
* 내 gitea project를 적당히 최소한의 구성으로 만들고 
* nodejs 를 거기를 바라보도록 설정해서 재설치
* mariadb도 함 붙여보기

# 삭제

기본으로 설치했다면 제거는 단순하게 가능함.
```
$ helm delete mynode        # 아까 설치할 때 그 이름. 혹시 기본 네임스페이스 설정 안했으면 -n <namespace> 옵션 추가. 

$ kubectl get pvc   # PV가 Retain 모드로 생성되었기에 안지워졌을 수 있음. 내용 보전한 채 다시 설치할 게 아니라면 직접 지워야 함. 
NAME             STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
mynode-mongodb   Bound    pvc-151663fa-4337-4b63-96f3-2ba1969c8dbf   8Gi        RWO            elastic-sc     59m
$ kubectl delete pvc mynode-mongodb
```
참, 이 문서에서는 배스천에서 설치하고 나서 PC로 넘어왔기에 삭제는 배스천에서 하는 식으로 적었는데, 사실 PC에서 설치/제거 작업을 완성할 수도 있음.
그럴 경우 PC에 helm repo 설정 다 해 주어야 함. (helm 이 윈도우에서 설치가 되는 것 같긴 한데 저도 안해봤습니다)

네임스페이스도 깔끔하게 지우려면
```
$ kubectl delete ns selee
```
