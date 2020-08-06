# Elasticsearch 시험 설치

이미 bitnami charts로 Elasticsearch가 설치되었지만 elastic.co 제공 차트로도 한 번 설치해 보자.

먼저 저장소 등록
```
$ helm repo add elastic https://helm.elastic.co
```

이후 설치..인데
다음 링크를 참조: https://hub.helm.sh/charts/elastic/elasticsearch

``` 
$ cat elasticsearch-values.yaml
clusterName: elasticsearch-test
nodeSelector:
  role: mgmt
persistence:
  enabled: true

$ helm install -n default elasticsearch --version 7.8.1 -f elasticsearch-values.yaml elastic/elasticsearch
NAME: elasticsearch
LAST DEPLOYED: Wed Aug  5 09:15:20 2020
NAMESPACE: default
STATUS: deployed
REVISION: 1
NOTES:
1. Watch all cluster members come up.
  $ kubectl get pods --namespace=default -l app=elasticsearch-test-master -w
2. Test cluster health using Helm test.
  $ helm test elasticsearch --cleanup
```

```
$ kubectl get po -n default
NAME                          READY   STATUS     RESTARTS   AGE
elasticsearch-test-master-0   0/1     Init:0/1   0          24s
elasticsearch-test-master-1   0/1     Init:0/1   0          24s
elasticsearch-test-master-2   0/1     Pending    0          24s  # 이건 노드 CPU/Mem 등이 모자라서 node scale up 을 하는 중이어서 대기중 

$ kubectl describe po -n default elasticsearch-test-master-0 | tail -5
  Type     Reason            Age        From                Message
  ----     ------            ----       ----                -------
  Warning  FailedScheduling  <unknown>  default-scheduler   0/4 nodes are available: 1 Insufficient memory, 2 node(s) didn't match node selector, 4 Insufficient cpu.
  Warning  FailedScheduling  <unknown>  default-scheduler   0/4 nodes are available: 1 Insufficient memory, 2 node(s) didn't match node selector, 4 Insufficient cpu.
  Normal   TriggeredScaleUp  99s        cluster-autoscaler  pod triggered scale-up: [{eksctl-mta-cluster-nodegroup-mta-mgmt-ng-NodeGroup-99D2CVRZ0VOS 2->3 (max: 3)}]
```

결국 자원문제로 다음과 같이 바꿔 설치함
```
$ cat elasticsearch-values.yaml
clusterName: elasticsearch-test
nodeSelector:
  role: mgmt
persistence:
  enabled: true
replicas: 1
```


이번엔 kibana 설치 차례임.
```
$ cat kibana-values.yaml
elasticsearchHosts: http://elasticsearch-test-master:9200
nodeSelector:
  role: mgmt

$ helm install -n default kibana --version 7.8.1 -f kibana-values.yaml elastic/kibana
NAME: kibana
LAST DEPLOYED: Wed Aug  5 09:28:41 2020
NAMESPACE: default
STATUS: deployed
REVISION: 1
TEST SUITE: None
```


