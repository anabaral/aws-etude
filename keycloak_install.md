# AWS에서 keycloak 설치 및 설정

제 저장소중 PC의 virtualbox 내에서의 keycloak 설치는 이미 설명한 곳이 있습니다. 그래서 필요 없을 거라 생각도 되지만
helm 을 이용한 설치를 하려고 보니 mariadb도 설치하는 등 많이 차이가 있고 theme 설정도 하게 되어 여기 설명을 붙이게 되었습니다.

이번에는 기본 db인 h2를 사용하지 않고 mariadb를 사용해 보려 합니다.
일단 저장소를 확인해 볼께요.
<pre><code>$ helm search repo stable | grep keycloak
stable/keycloak                         4.10.1          5.0.0                   DEPRECATED - Open Source Identity and Access Ma...
$ helm search repo stable | grep mariadb
stable/mariadb                          7.3.14          10.3.22                 DEPRECATED Fast, reliable, scalable, and easy t...
</code></pre>

둘 다 DEPRECATED 상태네요.. 다음 명령어로 다른 저장소를 추가합니다.
<pre><code>$ helm repo add bitnami https://charts.bitnami.com/bitnami  # for mariadb
$ helm repo add codecentric https://codecentric.github.io/helm-charts  # for keycloak</code></pre>


<pre><code>$ kubectl create ns keycloak</code></pre>

mariadb 먼저 설치해 보죠.
<pre><code>$ helm install keycloak-mariadb -n keycloak bitnami/mariadb --set global.storageClass=gp2 \
     --set db.name=keycloakdb  --set db.user=keycloak  --set db.password=keycloak
     --set master.persistence.mountPath=/opt/bitnami/mariadb

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
</code></pre>
출력 메시지가 있었던 것 같은데 갈무리를 까먹어서 생략.

realm을 추가하다 보면 새로운 로그인 테마를 설정할 수 있다는 사실을 깨닫습니다.
이를 위해 한 테마를 fork 해왔습니다. ( https://github.com/anabaral/alfresco-keycloak-theme )
테마를 만들고 설정하는 것은 쉬운데 재시작을 해도 유지가 되어야 하므로 (이걸 사내 연구프로젝트 같은 걸로 진행했는데 
비용 부담이 커서 매일 새벽시간에는 인스턴스를 꺼버립니다. https://github.com/anabaral/aws-etude/blob/master/instances.md 참조)
영속성을 부여해야 합니다.





