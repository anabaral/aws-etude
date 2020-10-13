# nodejs 어플리케이션 개발

[nodejs 개발연습 문서](https://github.com/anabaral/aws-etude/blob/master/nodejs-etude.md) 를 좀더 발전시켜 실 개발을 해보자.

소스는 https://gitea.skmta.net/selee/nodejs-etude 여기에서 관리했는데, stage-1...stage-10 까지 개발해 가면서 어느 정도 완성할 때마다 master로 합쳤음.

개발 과정에서 경험한 것들을 두서없이 나열하고 나중에 정리할 예정.


## 개발패턴



## mongodb

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
