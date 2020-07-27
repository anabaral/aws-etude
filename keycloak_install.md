# AWS에서 keycloak 설치 및 설정

제 저장소중 PC의 virtualbox 내에서의 keycloak 설치는 이미 설명한 곳이 있습니다. 그래서 필요 없을 거라 생각도 되지만
helm 을 이용한 설치를 하려고 보니 mariadb도 설치하는 등 많이 차이가 있고 theme 설정도 하게 되어 여기 설명을 붙이게 되었습니다.

이번에는 기본 db인 h2를 사용하지 않고 mariadb를 사용해 보려 합니다.

## 저장소 추가

일단 저장소를 확인해 볼께요.
<pre><code>$ helm search repo stable | grep keycloak
stable/keycloak                         4.10.1          5.0.0                   DEPRECATED - Open Source Identity and Access Ma...
$ helm search repo stable | grep mariadb
stable/mariadb                          7.3.14          10.3.22                 DEPRECATED Fast, reliable, scalable, and easy t...
</code></pre>

둘 다 DEPRECATED 상태네요.. 다음 명령어로 다른 저장소를 추가합니다.
<pre><code>$ helm repo add bitnami https://charts.bitnami.com/bitnami  # for mariadb
$ helm repo add codecentric https://codecentric.github.io/helm-charts  # for keycloak</code></pre>

## mariadb 설치
<pre><code>$ kubectl create ns keycloak</code></pre>

mariadb 먼저 설치해 보죠.
<pre><code>$ helm install keycloak-mariadb -n keycloak bitnami/mariadb --set global.storageClass=gp2 \
     --set db.name=keycloakdb  --set db.user=keycloak  --set db.password=keycloak
#     --set master.persistence.mountPath=/opt/bitnami/mariadb
#  마지막 한 줄은 불필요. 특정 버전의 이미지에서 문제가 되어 고쳤던 건데 현재는 오히려 기본값이어야 제대로 동작함 (스크립트에 박혀 있음) 

NAME: keycloak-mariadb
LAST DEPLOYED: Thu Jul  2 08:42:45 2020
NAMESPACE: keycloak
STATUS: deployed
REVISION: 1
NOTES:
Please be patient while the chart is being deployed

Tip:

  Watch the deployment status using the command: kubectl get pods -w --namespace keycloak -l release=keycloak-mariadb

Services:

  echo Master: keycloak-mariadb.keycloak.svc.cluster.local:3306
  echo Slave:  keycloak-mariadb-slave.keycloak.svc.cluster.local:3306

 Administrator credentials:

   Username: root
   Password : $(kubectl get secret --namespace keycloak keycloak-mariadb -o jsonpath="{.data.mariadb-root-password}" | base64 --decode)

 To connect to your database:

   1. Run a pod that you can use as a client:

       kubectl run keycloak-mariadb-client --rm --tty -i --restart='Never' --image  docker.io/bitnami/mariadb:10.3.23-debian-10-r44 --namespace keycloak --command -- bash

   2. To connect to master service (read/write):

       mysql -h keycloak-mariadb.keycloak.svc.cluster.local -uroot -p keycloakdb

   3. To connect to slave service (read-only):

       mysql -h keycloak-mariadb-slave.keycloak.svc.cluster.local -uroot -p keycloakdb

 To upgrade this helm chart:

   1. Obtain the password as described on the 'Administrator credentials' section and set the 'rootUser.password' parameter as shown below:

       ROOT_PASSWORD=$(kubectl get secret --namespace keycloak keycloak-mariadb -o jsonpath="{.data.mariadb-root-password}" | base64 --decode)
        JeugQ3vIdM
       helm upgrade keycloak-mariadb bitnami/mariadb --set rootUser.password=$ROOT_PASSWORD
</code></pre>
메시지가 길게 나오는데 일단 무시합시다. 서비스 이용을 위한 비번만 기억해 두면 됩니다. (ROOT_PASSWORD 관리법은 운영할 때는 필요하겠네요) 


## keycloak 설치

이제 keycloak을 설치할 건데, 지금껏 쌓아온 경험이 있으니 PVC 를 미리 만들어 두죠.
<pre><code>$ vi keycloak-pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: keycloak-storage-1
  labels:
    app: keycloak
  namespace: keycloak
spec:
  storageClassName: gp2
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi

$ kubectl apply -f keycloak-pvc.yaml
</code></pre>

mariadb 를 설치할 때는 <code>--set</code> 옵션을 활용했는데 keycloak 설치에는 값이 많으니 <code>values.yaml</code> 를 쓰겠습니다.
<pre><code>$ vi keycloak-values.yaml
keycloak:
  password: keycloak
  persistence:
    dbVendor: mariadb
    dbName: keycloakdb
    dbHost: keycloak-mariadb.keycloak
    dbPort: 3306
    dbUser: keycloak
    dbPassword: keycloak
  ingress:
    enabled: true
    annotations: { "kubernetes.io/ingress.class":"alb", "alb.ingress.kubernetes.io/scheme":"internet-facing", "alb.ingress.kubernetes.io/listen-ports":"[{\"HTTP\":80}, {\"HTTPS\":443}]", "alb.ingress.kubernetes.io/target-type":"ip"}
    hosts: ["keycloak.skmta.net"]
    path: "/*"
  serviceAccount:
    create: true
  extraVolumes: |
    - name: data
      persistentVolumeClaim:
        claimName: keycloak-storage-1

  extraVolumeMounts: |
    - mountPath: /opt/jboss/keycloak/standalone/data
      name: data
</pre></code>

준비가 되었으니 설치합니다.
<pre><code>helm install keycloak -n keycloak -f keycloak-values.yaml codecentric/keycloak
NAME: keycloak
LAST DEPLOYED: Mon Jul 27 05:55:27 2020
NAMESPACE: keycloak
STATUS: deployed
REVISION: 1
NOTES:
Keycloak can be accessed:

* Within your cluster, at the following DNS name at port 80:

  keycloak-http.keycloak.svc.cluster.local

* From outside the cluster, run these commands in the same shell:

  export POD_NAME=$(kubectl get pods --namespace keycloak -l app.kubernetes.io/instance=keycloak -o jsonpath="{.items[0].metadata.name}")
  echo "Visit http://127.0.0.1:8080 to use Keycloak"
  kubectl port-forward --namespace keycloak $POD_NAME 8080

Login with the following credentials:
Username: keycloak

To retrieve the initial user password run:
kubectl get secret --namespace keycloak keycloak-http -o jsonpath="{.data.password}" | base64 --decode; echo

</code></pre>


## optional: 테마 추가를 위한 설정

realm을 추가하다 보면 새로운 로그인 테마를 설정할 수 있다는 사실을 깨닫습니다.
이를 위해 한 테마를 fork 해왔습니다. ( https://github.com/anabaral/alfresco-keycloak-theme )
테마를 만들고 설정하는 것은 쉬운데 재시작을 해도 유지가 되어야 하므로 (이걸 사내 연구프로젝트 같은 걸로 진행했는데 
비용 부담이 커서 매일 새벽시간에는 인스턴스를 꺼버립니다. https://github.com/anabaral/aws-etude/blob/master/instances.md 참조)
영속성을 부여해야 합니다.

대충 절차는 이러합니다.
<pre><code>
$ vi keycloak-pvc.yaml # 위의 작성한 파일에 다음 내용 추가
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: keycloak-theme-storage-1
  labels:
    app: keycloak
  namespace: keycloak
spec:
  storageClassName: gp2
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
$ kubectl apply -f keycloak-pvc.yaml
persistentvolumeclaim/keycloak-theme-storage-1 created

$ kubectl get statefulset -n keycloak keycloak -o yaml > keycloak-sts.yaml
$ vi keycloak-sts.yaml # 적당한 위치로 마운트 하기 위해
... resourceVersion, uid 같은 항목 삭제하고 
spec:
  template:
    spec:
      containers:
      - command:
        ...
        name: keycloak
        ...
        volumeMounts:
        - mountPath: /opt/jboss/keycloak/themes_1 # 추가(임시설정)
          name: theme                             # 추가
      volumes:
        - name: theme
          persistentVolumeClaim:                  # 추가
            claimName: keycloak-theme-storage-1   # 추가

$ kubectl apply -f keycloak-sts.yaml
$ kubectl exec -it -n keycloak keycloak-0 -- bash # 컨테이너로 들어가서
bash-4.4$ cd /opt/jboss/keycloak/
bash-4.4$ cp -a themes/* themes_1/                # 원래 있던 내용 복사

$ vi keycloak-sts.yaml # 이번엔 제대로 된 위치로 마운트
...
        volumeMounts:
        - mountPath: /opt/jboss/keycloak/themes # 맞는 위치로 변경

$ kubectl apply -f keycloak-sts.yaml
</code></pre>
여기까지 하면 새 테마를 추가하기 위한 준비가 끝났습니다.

이제 테마를 추가해 보죠. 아래 예제는 제가 ci 라는 realm을 추가했다고 가정하고 진행한 것입니다.
<pre><code>
$ kubectl exec -n keycloak keycloak-0 -- mkdir /opt/jboss/keycloak/themes/ci
$ git clone https://github.com/anabaral/alfresco-keycloak-theme
$ cd alfresco-keycloak-theme/
$ kubectl cp ./theme/login keycloak/keycloak-0:/opt/jboss/keycloak/themes/ci
</code></pre>
여기까지 하면 keycloak UI 에 들어가서 ci realm 을 선택하고 Realm Settings 선택 - Themes 탭 선택 - Login Theme 항목에서 ci 를 선택할 수 있게 됩니다.
혹은 개별 client 설정에서 Settings 탭 선택 - Login Theme 항목에서 선택할 수도 있습니다.

선택하고 나면 달라진 테마를 볼 수 있습니다.
![두 테마 ](https://github.com/anabaral/aws-etude/blob/master/theme_compare.png)


## openldap 설치
keycloak 만 설치한다고 다가 아니죠. ldap도 설치해야 합니다. 지난 번 PC virtualbox 기반 설치때도 helm을 사용했지만 이번엔 좀더 나아가 봅시다.
영속성 부여도 아주 간단하게 됩니다.
<pre><code>$ helm install openldap  --namespace  keycloak --set persistence.enabled=true --set persistence.storageClass=gp2 stable/openldap </code></pre>

메시지가 길게 나올텐데 이것만 기억하면 됩니다:
<pre><code># 관리자 비밀번호 구하기: 
kubectl get secret --namespace keycloak  openldap -o jsonpath="{.data.LDAP_ADMIN_PASSWORD}" | base64 --decode; echo
</code></pre>

이걸 기억하기 까다로우면 위의 secret 내용을 다음과 같이 구해서 바꾸시면 됩니다:
<pre><code>$ echo 'change_password_like_this' | base64
<결과가텍스트로표시됨.이걸복사>

$ kubectl edit secret -n keycloak openldap # vi편집창에서 .data.LDAP_ADMIN_PASSWORD 항목을 찾아 복사한 텍스트로 교체
</code></pre>

참고로, openldap을 재설치하거나 사용자/그룹에 변경이 있는 경우 keycloak에서 자동으로 동기화가 되는 것 같지 않습니다.
keycloak 내의 각 realm의 User Federation 메뉴에서 [Synchronize all users] 같은 버튼으로 수동 동기화를 해줘야 합니다. 
자동으로 될 필요도 있을 것 같은데 아직 방법을 못찾았습니다.

여기까지 설치하면 openldap 에 사용자 등록하는 건 ldapadd 같은 툴을 사용하면 될 텐데, 배스티온 서버가 클러스터 내부 서버가 아니다 보니
직접 접근해 관리하려면 NodePort를 열거나 Ingress를 만들거나.. phpldapadmin 같은 툴을 설치해야 합니다.
여기서는 툴을 설치해 사용해 보죠. 

지금까지 있던 helm 저장소에 phpldapadmin이 없네요.
<pre><code>$ helm repo add cetic https://cetic.github.io/helm-charts  # for phpldapadmin</code></pre>
설치합니다.
<pre><code>$ helm install  phpldapadmin --namespace keycloak cetic/phpldapadmin --set env.PHPLDAPADMIN_LDAP_HOSTS=openldap.keycloak</code></pre>
이제 브라우저로 여기 붙어야 하는데 ingress 를 새로 만들려면 다음 설정을 사용합니다:
<pre><code>$ cat phpldapadmin-ing.yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  annotations:
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP":80}, {"HTTPS":443}]'
    alb.ingress.kubernetes.io/scheme: internet-facing
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/target-type: ip
    meta.helm.sh/release-name: phpldapadmin
    meta.helm.sh/release-namespace: keycloak
  generation: 1
  labels:
    app: phpldapadmin
    app.kubernetes.io/managed-by: Helm
    chart: phpldapadmin-0.1.4
    heritage: Helm
    release: phpldapadmin
  name: phpldapadmin
  namespace: keycloak
spec:
  rules:
  - host: phpldapadmin.skmta.net
    http:
      paths:
      - backend:
          serviceName: phpldapadmin
          servicePort: http
        path: /*
status:
  loadBalancer: {}
</code></pre>

이렇게 하면 AWS 내에 ALB(=application load balancer)가 생성되죠. 여기 규칙을 잘 다듬은 후 Route53 에서 이 ALB를 지정하도록 설정하면 접속이 될 겁니다.
그런데 이게 문제가 하나 있습니다. 다름아닌 비용인데요..
ingress 하나 추가할 때마다 ALB가 하나씩 늘어나는데, 이 ALB가 존재하는 것만으로도 월 몇십 불 정도의 비용을 부과합니다. 
대형 프로젝트라면 모를까 꼬꼬마 연구 프로젝트에선 이 비용도 큰 부담이라 살펴보니 같은 namespace 안에서는 ingress를 하나만 쓰면서도 host를 분리해 쓸 수 있더군요.
그래서 위의 ingress 설정은 keycloak ingress 설정과 통합하게 됩니다.
<pre><code>$ cat keycloak-ing.yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  annotations:
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP":80}, {"HTTPS":443}]'
    alb.ingress.kubernetes.io/scheme: internet-facing
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/target-type: ip
    meta.helm.sh/release-name: keycloak
    meta.helm.sh/release-namespace: keycloak
  creationTimestamp: "2020-07-02T09:59:15Z"
  generation: 1
  labels:
    app.kubernetes.io/instance: keycloak
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/name: keycloak
    app.kubernetes.io/version: 10.0.0
    helm.sh/chart: keycloak-8.2.2
  name: keycloak
  namespace: keycloak
spec:
  rules:
  - host: keycloak.skmta.net
    http:
      paths:
      - backend:
          serviceName: keycloak-http
          servicePort: http
        path: /*
  - host: phpldapadmin.skmta.net
    http:
      paths:
      - backend:
          serviceName: phpldapadmin
          servicePort: http
        path: /*
</code></pre>


