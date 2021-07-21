# Jenkins 에서 Pipeline 설정

이 문서는 2021-07 작업을 기반으로 합니다. 몇 가지 가정이 있는데 문서에 보일 것입니다.


## 주의사항

당신이 2021년에 이 문서를 보고 있다면 염두에 두세요:

- 제발 Bitnami의 Jenkins를 설치하지 마세요.  
  겉으로는 그럴듯 한데 뭔가 심각하게 빠져 있어서 Pipeline 설정할 때 뇌가 뽀개져 버릴 겁니다.  
  특히 kubernetes pod 로 jenkins agent를 만들때 필요한 준비가 제대로 안 되어 있습니다.
- Azure Kubernetes 즉 AKS 를 사용하려고 생각한다면 재고해 보세요.
  Azure 기본 storageclass 들이 제공하는 스토리지는 엄밀히 말해 Linux/Unix 파일시스템이 아닌 것 같습니다.
  읽고 쓰는 건 얼추 되는데 무슨 퍼미션 같은 문제가 갑자기 튀어나와 우리 뇌를 휘젓습니다.
  아주 안되는 것도 아니고 다 되는 것도 아니고... 이런 것 가지고 수명을 단축시키지 마시길.

(2022년쯤이면 혹시 나아질 지도 모르겠습니다)


## Jenkins 설치

다음 명령으로 설치합니다.

```
$ helm repo list | grep jenkins
jenkinsci                               https://charts.jenkins.io

$ vi jenkins-values.yaml
controller:
  adminUser: admin
  adminPassword: ~아싸라비아콜롬비아~
  ingress:
    enabled: true
    hostName: jenkins.sk-az.net
    annotations:             # 여기 annotation들은 cert-manager를 위한 설정인데 여기선 설명 안함
      cert-manager.io/cluster-issuer: "letsencrypt-prod"      
      appgw.ingress.kubernetes.io/ssl-redirect: "true"
      kubernetes.io/ingress.class: nginx
      kubernetes.io/tls-acme: "true"
    tls:               # 아 이걸 빠뜨렸었네.. 어쩐지 tls가 설정 안되더라고.. 저도 안해봐서 이건 설명 안합니다
serviceAccount:
  name: jenkins
serviceAccountAgent:   # 이거 만들라고 명시한건데 안만들었네요
  name: jenkins-robot
  create: true         # 아 이걸 빠뜨렸었구나.. 좀 잘 볼 걸
$ helm install jenkins jenkinsci/jenkins -n ca  -f jenkins-values.yaml
```

`jenkins-values.yaml` 파일 작성을 위한 참조: https://github.com/jenkinsci/helm-charts/blob/main/charts/jenkins/VALUES_SUMMARY.md


## 추가 설정

### Ingress

TLS가 설정이 빠져서 추가 설정합니다:
```
$ kubectl edit ingress -n ca jenkins
...
  tls:                  # 적절한 위치가 어딘지는 구글링으로 찾아보세요
  - hosts:
    - jenkins.sk-sz.net
    secretName: jenkins-tls
...
```

### PVC

PersistentVolumeClaim 을 추가합니다:
```
$ vi ca-workspace-pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  annotations:
    volume.beta.kubernetes.io/storage-provisioner: kubernetes.io/azure-file
  finalizers:
  - kubernetes.io/pvc-protection
  name: ca-workspace
  namespace: ca
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 4Gi
  storageClassName: azurefile   # 이것 가지고 고생함
  volumeMode: Filesystem
```
`storageClassName` 이 중요한데, 우리가 쓸 수 있는 게 대략 
- `default` 하고 `azurefile` 중에 하나를 택하거나 
- 이 둘 중 하나를 기반으로 커스텀 생성하는 것입니다.

그런데 여러 시도가 무색해져서 그냥 순정(?)으로 택했습니다.

참고로 뭘 잘못했는 지는 모르지만 퍼미션 문제로 골치 썩입니다. 결국 이해한 것은
- 뭔지 모르나 퍼미션 문제를 일으키는 Operation이 존재함. (read나 write는 아닌 것 같음. 777이래도 에러가 나곤 함)
- 그 Operation은 파일/디렉터리의 소유자 (혹은 그룹) 만이 실행할 수 있고
- Linux/Unix 에서는 root는 그걸 무시해야 하는데 root도 지배받음. (테스트해 보면 root 가 chmod /chgrp을 못함)
- azurefile 에서 경험한 것인데, default(azure disk) 에서도 미묘하게 다르지만 비슷한 문제를 겪음.
- 이건 문서상으로 봤던 건데, Azure 제공 스토리지에서 symbolic link 혹은 hard link 만드는 게 잘 안된다고 함. (둘 중 하나는 될 지도 모름)













