# 각종 쉘 스크립트 

유용하게 쓰였거나 혹은 나중에 참조할 일이 있는 스크립트를 정리.

## PersistentVolume 체크 스크립트

현재 클러스터 안에 존재하는 PV 들을 모두 체크해 이를 사용하는 Statefulset 과 Deployment 들을 찾고 이들의 replicaset 값을 얻어
이들을 모두 합함.  
```
#!/bin/sh
kubectl get pv --no-headers | awk '{split($6, a, "/") ; print (a[1] " " a[2]) }' | while read ns pvc
do
  kubectl get sts,deploy -n $ns --no-headers | awk '{split($1, a, "/") ; print (a[2]) }' | while read name
  do
    #echo "ns=$ns , pvc=$pvc , name=$name"
    pvc_cnt=$( kubectl get sts,deploy -n $ns $name -o yaml  2>/dev/null | grep $pvc | wc -l )
    #echo "pvc_cnt=$pvc_cnt"
    if [ $pvc_cnt -gt 0 ];then
      echo "ns=$ns , pvc=$pvc , name=$name , pvc_cnt=$pvc_cnt" 1>&2
      kubectl get deploy,sts -n $ns $name -o jsonpath="{.items[*].spec.replicas}" 2>/dev/null
      echo ""
    fi
  done
done | awk '{sum+=$1}END{print "sum=" sum}'
```  
...근데 이걸 왜 했지?

... 원래 목적은 PV 를 사용하는 프로세스들의 총 개수를 알아보려 했던 것 같음.



