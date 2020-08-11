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

### keycloak 에 js-console 등록

gitea에 등록할 만만한 프로젝트로 js-console 을 사용해 보겠습니다.

keycloak 에 dev realm이 없다면 추가하고, 그 안에 js-console 이라는 client를 생성합니다.
Settings 탭에서 다음을 입력해 줍니다:
- Client ID, Name : js-console
- Login Theme: base
- Client Protocol: openid-connect
- Access type: public (이게 confidential이면 CORS 문제가 불거지는데 어떻게 해결하는지 아직 모릅니다. 
  처음에 js-console apache 설정에 ```Header set Access-Control-Allow-Origin: *``` 추가하는 걸로 접근했었는데 실제론 keycloak 에서 거부하는 거라 답이 아닌 것 같네요)
- Root URL: http://js-console.skmta.net
- Redirect URL: http://js-console.skmta.net/*
- Admin URL: http://js-console.skmta.net
- Web Origins: http://js-console.skmta.net (이게 keycloak에서 CORS 문제에 대처하는 데 쓰는 허용 URL입니다. 
  이걸 제대로 입력해 줘도 Access type이 confidential이면 실패)

keycloak의 CORS 이슈 관련해서는 아래 그림에 간단히 적었습니다. 상세한 설명은 저도 충분히는 모르고 구글링 하면 자료가 많으니 생략할게요.
![CORS Issue](https://github.com/anabaral/aws-etude/blob/master/keycloak_CORS.png)

Installation 탭에서 
- Format Option을 'Keycloak OIDC Json' 으로 선택하고 나오는 json 텍스트를 갈무리 해 둡니다.
## gitea에 샘플 프로그램 js-console 등록

이제 js-console을 gitea에 붓습니다.

먼저 배스티온에서 소스를 구해오고 git 정보를 지웁니다
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

  </code></pre>

- index.html
  <pre><code>https://keycloak.k8s.com:32443/... --> https://keycloak.skmta.net/...</code></pre>
- keycloak.json
  . keycloak 의 client 설정에서 갈무리해 둔 json 내용을 부으면 됩니다.

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
다음과 같이 설정합니다:
* 로그인 후 홈 화면에서 '새로운 Item' 선택
* item 이름을 'js-console' 로 정하고 유형은 Pipeline 으로 선택
* 구성화면이 나오면
  - General 항목은 모두 비우고
  - Build Triggers 항목에선 PollSCM을 체크, 내용은 채우지 말고
  - Pipeline 항목은 'Pipeline Script from SCM' 을 선택 후
    . SCM = Git, 
    . Repositories = https://gitea.skmta.net/skmta/js-console.git , 
    . Credential은 본래 줘야 하는데, 현재는 Gitea가 anonymous read를 허용하므로 비워둠
    . Scriptpath = Jenkinsfile
    . 저장, Apply
위에서 PollSCM을 체크하고 내용을 채우면 그 내용에 따라 Poll을 하는 반면 내용을 비워두면 Gitea Webhook 설정에 의해 "깨워질 때" 체크아웃이 수행됩니다.

## gitea webhook 설정

js-console 프로젝트에 가면 (그 관리권한을 가지고 있다면) '설정'메뉴에 접근할 수 있습니다.

Webhook이라는 탭에 들어가서 'Webhook 추가' - 'Gitea' 를 선택합니다.
* Target URL = https://jenkins.skmta.net/gitea-webhook/post?job=js-console
* HTTP Method = POST
* POST Content Type = application/json
* Trigger On = Push Events
* Active 체크

이제 해당 프로젝트에 어떤 변경이 생길 때마다 jenkins build가 가동될 것입니다. 

## ingress (js-console 접속용) 설정

빌드배포가 되는 것과 js-console 앱에 접근되는 것은 별개입니다. 적절한 인그레스 설정을 해 주어야 합니다.
js-console은 mta-dev 네임스페이스에 디플로이 되는데, 여기 이미 기존에 사용중인 인그레스가 있으므로 여기에 통합시키도록 하죠.
다음을 "잘" 삽입합시다.
<pre><code>$ kubectl edit ing -n mta-dev __<기존_ingress_이름>__
...
  - host: js-console.skmta.net
    http:
      paths:
      - backend:
          serviceName: js-console
          servicePort: 8000
...
</code></pre>

## Troubleshooting

Troubleshooting 은 경우에 따라 이걸 읽는 사람에게 해당 될 수도 있고 안될 수도 있는 문제해결 케이스입니다.

### 클라우드 설정에서 jenkins-agent 설정

처음 jenkins를 설치했을 때는 생기지 않던 문제가 갑자기 생겼습니다. 증상은 파이프라인에서 실행이 무한히 지연되는 것이었는데,
로그를 보면 Jenkinsfile 수행할 때 jnlp 컨테이너가 뜨자마자 다시 죽습니다. (위의 Jenkinsfile 을 보면 알겠지만 jnlp 컨테이너는
제가 선언하지 않았지만 기본으로 뜹니다) 진짜 문제의 원인을 알려면 jnlp 컨테이너 로그를 들여다 봐야 하는데 이게 너무 금방 죽어서
제대로 얻지 못하거나 로그가 나와도 '이게 전부야? 더 없어?' 식으로 파악이 안되었습니다. 

결국 여러 가지 꼼수로 얻어낸 결론은 다름아닌 설정상의 문제였습니다.
![jenkins_agent_config](https://github.com/anabaral/aws-etude/blob/master/configure-clouds-jenkins-agent.png)
위에서처럼 URL 형식이 아니어야 할 입력값을 URL 형식으로 넣었다고 에러가 났었습니다.

그럼 처음 설치했을 때는 왜 괜찮았을까?

현재 설치 중인 jenkins 차트는 jenkins를 항상 최신 이미지로 설치하게 되어 있습니다. 처음 설치했을 때는 'configuration as code' 기능이 포함된 채의
jenkins였고요. 제 다른 글을 보면 아시겠지만 그 기능 때문인지 그 기능에 맞춘 설정의 부작용 때문인지 Jenkins UI에서 설정했던 값들이 POD 재시작 시
<code>jenkins-jenkins-jcasc-config</code> ConfigMap 의 내용에 의해 덮어써집니다.
이걸 해결해 주기 위해서는 UI에서 설정했던 내용에 대응되는 해당 ConfigMap의 내용을 같이 고쳐줘야 했습니다.<br>
다행히도 엊그제 설치한 jenkins는 이 기능이 빠져 있었습니다. 대신에 그 ConfigMap에 있던 내용의 JenkinsTunnel 같은 몇몇 항목이 빈간으로 남아있게 되었고
시행착오 끝에 그 항목을 채우기는 했지만 형식이 맞지 않았던 겁니다.
