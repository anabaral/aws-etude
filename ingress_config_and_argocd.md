# Ingress 설정을 ArgoCD와 연계하여

제목이 좀 이상하지만 사연이 있습니다. 우리는 AWS 위에 EKS를 구축하고 Ingress를 설정하여 사용하고 있습니다.
그런데 아시다시피 Ingress는 다음과 같은 특징이 있습니다.

* 네임스페이스 별로 하나 이상의 Ingress 설정이 가능합니다.
* AWS에서 Ingress 하나 당 ALB (Application Load Balancer) 가 하나씩 만들어집니다.
* 문제는 **AWS는 ALB 하나 당 과금을 한다는 점입니다** (!!!) ALB의 과금량은 사용량을 무시할 때 월 몇십불 수준이므로 대규모 프로젝트에서는 무시할 수도 있겠지만 
  개인이나 소규모 단위 프로젝트에서는 부담됩니다.

이걸 해결해 주기 위해 우리는 '같은 네임스페이스의 Ingress들은 하나로 합치자' 는 결론에 도달했습니다.

그런데 설정 방식은 맞춘다 해도 다른 문제가 하나 남습니다.
우리가 보통 Ingress를 바꾸는 방식은 다음과 같습니다:
```
$ kubectl get ingress -n our_namespace our_ingress -o yaml > our_ingress_ing.yaml
$ vi our_ingress_ing.yaml
$ kubectl apply -f our_ingress_ing.yaml
```
그리고 한 번 바꿀 때 생성되는 파일을 고쳐가면서 적용하게 되죠.

이 방식의 장점은 파일로 중간단계가 남고 이를 파일명 변경 보관 등 활용해 히스토리 관리나 백업 등이 용이합니다.

그러나 여러 사람들이 고칠 때는 이게 당연히 문제가 됩니다.
내가 고친 ing 변경분이 어느 순간 남에 의해 덮어써질 수 있습니다.

덮어쓰이는 문제를 해결하는 방법 중 하나는 다음과 같은 직접 변경인데
```
$ kubectl edit ingress -n our_namespace our_ingress
[ vi editor 가 열려 편집 가능 ]
```
이건 히스토리 관리 등 여러 가지를 포기해야 합니다.

그런데 생각해 보니... ArgoCD가 이걸 위해 존재하지 않았던가!

방법은 간단합니다.

* git 혹은 gitea에 ingress 파일을 포함한 프로젝트를 추가하고
* 프로젝트에 ingress 파일을 가져다 두고
* ArgoCD 에 프로젝트 및 APP 을 추가해 이 파일을 Sync 하게 합니다.
* 이후로는 git 을 건드리면 바로 반영될 겁니다. 
* 버전관리도 되니까 안심.

