# 배포를 위한 jenkins 와 gitea 의 설정

gitea 는 제가 직접 설치하지 않아서 설치 과정은 생략합니다. 다만 몇 가지 특징만 적어두겠습니다.
- gitea 는 기본설정으로 시작하면 회원가입 요청을 하면 그냥 사용자가 생성됩니다. 
  (이걸 막으려면 관리자가 Disable Self-Registration 옵션을 켜야 함)
- 설치 후 가장 최초 생성된 사용자가 사이트 관리자가 됩니다(!).
- 이후 관리자가 다른 사용자들을 관리자로 지정할 수 있습니다.

## gitea keycloak 인증 설정

gitea 가 keycloak 인증으로 로그인 가능케 하려면 관리자가 다음을 설정해야 합니다:
- gitea ingress 설정이 되어 https://gitea.skmta.net 으로 접속 된다고 가정
- keycloak에서
  * ci realm 에 client 추가 : gitea
  * 이 때 url을 https://gitea.skmta.net 로 지정
  * Settings 탭에서 
    - Access Type : confidential 로 지정, 이렇게 하면 Credentials 탭이 나옴
    - client id 복사해 둠
  * Credentials 탭에서
    - Client Authenticator : client id and secret
    - secret 아이디를 복사해 둠
- gitea에서
  * Site Administration 메뉴 - Authentication Sources 탭 - [Add Authentication Source] 버튼
  * 인증이름: keycloak
  * OAuth2 Provider: OpenID Connect
  * client id, secret 입력
  * auto discovery url : https://keycloak.skmta.net/auth/realms/ci/.well-known/openid-configuration

이렇게 하고 로그인 화면에서
![login_form](https://github.com/anabaral/aws-etude/blob/master/gitea_openid_connect_form.png)
아래의 버튼을 클릭하여 keycloak 제공하는 로그인 화면으로 넘어가면 성공.

## gitea에 샘플 프로그램 등록

gitea에 등록할 만만한 프로젝트로 js-console 을 사용해 보겠습니다.

일단 배스티온에서 소스를 구해오고 git 정보를 지웁니다
<pre><code>$ git clone https://github.com/anabaral/keycloak-containers-demo/tree/master/js-console 
$ rm -rf ./.git
</code></pre>

이제 gitea에 commit/push 해봅시다.
<pre><code>$ git init
$ git add .
$ git commit -m 'initial'
$ git remote add origin https://gitea.skmta.net/skmta/js-console
$ git push origin master
</code></pre>

물론 이게 다가 아니고.. 다음을 고쳐야 합니다.
- Dockerfile
  <pre><code>
  FROM httpd

  ADD src /usr/local/apache2/htdocs/

  RUN (echo '' \
    && echo ''  \
    && echo '<VirtualHost *:80>'  \
    && echo '    Header set Access-Control-Allow-Origin "*"' \
    && echo '</VirtualHost>' ) >> /usr/local/apache2/conf/httpd.conf
  </code></pre>
  다른 건 달라질 게 없고, 웹페이지에서 다른 도메인의 사이트 (keycloak 사이트)로부터 몇몇 파일을 호출해야 하는데 
  그게 브라우저에 따라서는 CORS 위반으로 안받아집니다. 이걸 허용하려면 서버에서 "괜찮아 헤더" 를 보내줘야 합니다.
- index.html
  <pre><code>https://keycloak.k8s.com:32443/... --> https://keycloak.skmta.net/...</code></pre>
- keycloak.json
  . keycloak 의 client 설정에서 얻을 수 있는 json 내용을 부으면 됩니다.

이제 다음만 작성하면 끝(?)입니다.
- Jenkinsfile 작성 (빌드 파이프라인)
- js-console Deployment 파일 작성
- js-console Service 파일 작성

## Jenkinsfile 작성

<pre><code>def label="jenkins-${UUID.randomUUID().toString()}"


def this_time = new java.text.SimpleDateFormat("yyyyMMdd_HHmmss")
                .format(Calendar.getInstance(
                        TimeZone.getTimeZone("KST"), Locale.KOREA).getTime()
                  )

podTemplate(label:label, serviceAccount: "jenkins-robot", namespace: "mta-cicd",
  containers: [
    containerTemplate(name: 'docker', image: "docker:19.03", ttyEnabled: true, command: 'cat'),
    containerTemplate(name: 'kubectl', image: "lachlanevenson/k8s-kubectl:v1.16.12", ttyEnabled: true, command: 'cat')
  ],
  volumes: [
    hostPathVolume(hostPath: '/var/run/docker.sock', mountPath: '/var/run/docker.sock'),
    hostPathVolume(hostPath: '/var/lib/docker', mountPath: '/var/lib/docker'),
    hostPathVolume(hostPath: '/etc/hosts', mountPath: '/etc/hosts')
  ],
  workspaceVolume: persistentVolumeClaimWorkspaceVolume(claimName: "cicd-workspace", readOnly: false)
) {


    node(label){
      stage('Checkout') {
        checkout scm
      }
      stage('Build'){
        container('docker') {
            sh "docker build -t demo-js-console . "
            sh "docker tag demo-js-console harbor.skmta.net/mta-dev/demo-js-console:${this_time}"
        }
      }
      stage('Push'){
        container('docker') {
          try {
            withDockerRegistry(credentialsId: "0edcba68-3833-446e-93ef-4c6e8e7b9723", url: "http://harbor.skmta.net") {
              sh "docker push harbor.skmta.net/mta-dev/demo-js-console:${this_time}"
            }
          }catch(Exception e){
            echo e.toString()
            throw e
          }
        }
      }
      stage('Deploy'){
          container('kubectl') {
            try {
              sh "kubectl apply -f js-console-deploy.yaml"
              sh "kubectl set image -n mta-dev deployment/js-console js-console=harbor.skmta.net/mta-dev/demo-js-console:latest"
              sh "kubectl apply -f js-console-svc.yaml"
            }catch(Exception e){
              echo e.toString()
              throw e
            }
          }
      }
    }

}
</code></pre>
아, 제가 말을 안했던가요? 클러스터용 private docker registry 가 있어야 해서 harbor 를 설치했었습니다.  
그리고 위 내용을 보면 아시겠지만 아래 작성되는 js-console-deploy.yaml 과 js-console-svc.yaml 파일은 위 빌드배포 파이프라인에서 사용됩니다.

## Deployment, Service 파일 작성

js-console-deploy.yaml
<pre><code>apiVersion: apps/v1
kind: Deployment
metadata:
  annotations:
  generation: 1
  labels:
    app: js-console
    release: js-console
  name: js-console
  namespace: mta-dev
spec:
  minReadySeconds: 5
  progressDeadlineSeconds: 600
  replicas: 1
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      app: js-console
      release: js-console
  strategy:
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 25%
    type: RollingUpdate
  template:
    metadata:
      labels:
        app: js-console
        release: js-console
    spec:
      containers:
      - image: harbor.skmta.net/mta-dev/demo-js-console:latest
        imagePullPolicy: Always
        name: js-console
        ports:
        - containerPort: 8000
          protocol: TCP
        resources: {}
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      schedulerName: default-scheduler
      #securityContext:
      #  fsGroup: 1000
      #  runAsUser: 1000
      terminationGracePeriodSeconds: 30
status:
</code></pre>

js-console-svc.yaml
<pre><code>apiVersion: v1
kind: Service
metadata:
  annotations:
  labels:
    app: js-console
  name: js-console
  namespace: mta-dev
spec:
  ports:
  - name: http
    port: 8000
    nodePort: 30080
    protocol: TCP
    targetPort: 80
  selector:
    app: js-console
    release: js-console
  type: NodePort
</code></pre>

## jenkins 프로젝트 설정



## ingress (js-console 접속용) 설정
