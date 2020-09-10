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
- 기본적으로 설치하면 자기가 가진 git container (git 클라이언트 역할)로 github에서 샘플 앱을 받아 실행하는 것 같음.
  직접 소스를 안에 올리는 게 아니라 git으로 가져간다고? 신박한데.. 그럼 최종적으로는 gitea에 우리의 소스를 가져다 놓고 이를 등록해야 한다는 얘기.
당장 선택은 하지 말고 그냥 깔아보자.
```
$ helm install mynode bitnami/node
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



