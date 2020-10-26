# nodejs 어플리케이션 개발

[nodejs 개발연습 문서](https://github.com/anabaral/aws-etude/blob/master/nodejs-etude.md) 를 좀더 발전시켜 실 개발을 해보자.

소스는 https://gitea.skmta.net/selee/nodejs-etude 여기에서 관리했는데, stage-1...stage-10 까지 개발해 가면서 어느 정도 완성할 때마다 master로 합쳤음.

개발 과정에서 경험한 것들을 두서없이 나열하고 나중에 정리할 예정.


## 개발패턴

다음은 일반적인 웹 개발패턴 두 가지를 서술한 다이어그램이다:
![웹개발패턴](https://github.com/anabaral/aws-etude/raw/master/img/web_dev_diagram.svg)

보통 Java 개발이면 SpringFramework 와 JSP 를 사용하여 첫번째 패턴 위주로 개발하고 부수적으로 두번째 패턴을 활용한다.  
하지만 Nodejs의 경우 템플릿 개발이 약해서 ([EJS](https://github.com/mde/ejs) 같은 게 있긴 하지만) API 형태로 개발하게 되므로 두번째 패턴 위주로 개발하기로 하였다.  
이렇게 하면 서버사이드와 클라이언트사이드가 모두 Javascript 로 구현되므로 두 언어 이상 익숙해질 필요 없는 장점도 있다 (대신 단점도 수두룩)

## 형상관리

여기서의 개발은 전형적이지 않은 패턴을 취했다.  
Git에 적용하고 어플리케이션을 재시작 (혹은 rollout restart) 하면 어플리케이션이 시작할 때 git에서 읽어 실행한다.  
이미지를 만들지 않고 개발하는 셈인데 장단점이 있다.
- 별도의 이미지 생성 불필요
- 별도의 빌드 절차 생성/실행 불필요
- 간편하게 관리 가능
- (단점) 소스 양에 따라 다르지만 이미지로 만들어 띄우는 것보다 초기화 시간이 더 걸림. 

마지막 단점 때문에.. 운영에서는 이미지를 만들고 이를 띄우는 게 더 권장된다.  
다만 나는 여기서는 운영도 같은 패턴을 적용했다. 편해서..

```
[selee nodejs-etude] $ git branch stage-11     # 새 브랜치 따기
[selee nodejs-etude] $ git checkout stage-11   # 새 브랜치로 체인지
[selee nodejs-etude] $ (소스 개발...)
[selee nodejs-etude] $ git add .
[selee nodejs-etude] $ git commit -m '어쩌구 저쩌구 했음'
[selee nodejs-etude] $ git push origin stage-11

[selee nodejs-etude] $ kubectl delete po mynode-xxxxxxxx-yyyyy  # 현재 떠 있는 pod 죽이기
... 기다렸다가 결과 확인
... 개발 반복

... 완성되었다 생각되면
[selee nodejs-etude] $ git checkout master
[selee nodejs-etude] $ git merge stage-11
... vi 창이 나오는데 달리 할 게 없으면 그냥 나가면 됨
... merge 완료 결과 확인
[selee nodejs-etude] $ git add .
[selee nodejs-etude] $ git commit -m '어쩌구 저쩌구 했음'
[selee nodejs-etude] $ git push origin master

[selee nodejs-etude] $ kubectl rollout restart -n mta-infra deploy mta-admin-node
... 기다렸다가 결과 확인
```


## mongodb

mongodb 이용은 처음 접한 예제에 있었고 구글링 검색결과로도 강력한 툴로 소개되고 있는 mongoose를 별 고민 없이 채택.  

### not master and slaveOk=false 에러

이건 초반에는 나지 않다가 아무래도 개발하는 환경이 돈을 아낀다고 저녁~새벽 시간대에 껐다가 (k8s 노드들의 autoscaling group을 0로 세팅) 
다시 켜는 과정 + Persistent Volume 에 대한 Quota가 찬 것과 관련이 되어서 갑자기 그런 현상이 생겼다.

mongoose로 요청을 하면 응답 json에 저 메시지가 나오는데.. 찾아보니 접속은 마스터에만 해야 하고 마스터가 죽을 경우에만 슬레이브로 붙는 거니
평소에는 접속 인스턴스를 지정해 주는 것이 좋다고 함. 이게 정답인 지는 불확실하지만 일단 시도해 봄.
```
$ kubectl edit deploy -n mta-infra mta-admin-node
...
        - name: DATABASE_CONNECTION_OPTIONS
          value: replicaSet=rs0
```

## locale with mongoose

해프닝으로 끝나긴 했는데, 우리가 mongodb에 저장할 때 ```_id``` 필드만 빼곤 모두 넣는 형식이 자율적이다. 즉 아무 타입이나 가능함.

다만 ```date``` 필드의 모델을 정할 때 아무 생각 없이 ```Date``` 타입이라고 간주했고 그렇게 세팅했었는데 
데이터 저장 구현은(분업을 했었기에) ```String``` 타입으로 그냥 KST 기준 ```yyyy-MM-dd HH:mm:SS``` 형식만 맞춰 저장하는 거였음.
```
## 처음 구현
module.exports = mongoose.model('Alert', {
    date: {
        type: Date
    },
...
});
```

그런데 mongoose는 모델에서 ```Date``` 타입으로 정했으면 값을 넣거나 뺄 때 이를 UTC 기준으로 자동 매핑해 버림. 
그래서 mta-admin에서 리턴할 때 자동으로 UTC라고 간주하고 리턴했고, 이를 본 시각으로 환산하니 여지없이 +9 되었던 것. 

한편 데이터 저장하는 기능의 개발자와 데이터를 화면에 보여주는 개발자가 달랐기에 이게 잘못되었는 줄을 한참 후에야 알게 되었음..
문제의 원인도 조금 지나서야 이해하게 되었고..
```
## 나중 구현
## 
module.exports = mongoose.model('Alert', {
    date: {
        type: String
    },
...
});
```
사실 글로벌 기준으로는 ```Date``` 타입으로 매핑하는 게 맞는데..  
연구 프로젝트고 시간이 촉박한데다 저장 맡으신 분 업무가 과중해서 그냥 내가 맞춤.

