# jenkins 설치 (keycloak 설정과 함께)

jenkins를 설치하겠습니다. 일전에도 PC 의 virtualbox 기반 k8s 에서 설치한 적이 있는데 비슷하지만 약간 다른 방식으로 진행할 것입니다.

## 설치 설정 작성

ingress 등 들어가야 하는 항목이 복잡하여 values.yaml 파일을 먼저 작성하겠습니다.
<pre><code>$ vi jenkins-values.yaml
namespaceOverride: mta-cicd
master:
  adminPassword: <정해둔_비번>
  installPlugins: ["credentials-binding:1.23"]
  additionalPlugins: ["keycloak:2.3.0", "gitea:1.2.1", "docker-workflow:1.23", "kubernetes:1.25.7", "matrix-auth:2.6.2" ]
  ingress:
    enabled: true
    hostName: jenkins.skmta.net
    annotations: { "kubernetes.io/ingress.class":"alb", "alb.ingress.kubernetes.io/scheme":"internet-facing", "alb.ingress.kubernetes.io/listen-ports":"[{\"HTTP\":80}, {\"HTTPS\":443}]", "alb.ingress.kubernetes.io/target-type":"ip"}
    path: "/*"
persistence:
  storageClass: elastic-sc
</code></pre>
내용을 보자면,
- jenkins 설치할 때 몇 가지 plugin을 같이 설치합니다.
  - keycloak 플러그인을 미리 설치해 두는 게 좋습니다.
  - credentials-binding 은 이미 1.22 가 설치되어 있던데.. 1.23으로 업글합니다.
  - matrix-auth 는 권한관리를 위한 플러그인입니다. 
    이걸 설치하지 않고 선택할 수 있는 최선의 권한관리 옵션은 legacy 입니다만, 
    익명사용자가 정보를 마음대로 볼 수 있기 때문에 설치를 하는 편이 좋겠죠.
    matrix-auth 플러그인 사용은 직관적이므로 따로 설명하지는 않겠습니다.
- 관리자 비번을 미리 정합니다. 이 방식은 장단점이 있으므로 좀 생각해 봅시다: 
  - 정하지 않으면 랜덤방식으로 결정되는데 이게 가시성이 안좋아서 불편합니다.
  - 그러나 정하면, kubeapps 같이 설치된 charts 정보를 보여줄 때 이 values.yaml 내용이 평문으로 보입니다. 좀 불안하죠?
  - 허나 keycloak 인증방식을 사용하게 되면 어차피 최초 로그인때만 쓰이는 관리자이므로 큰 문제는 없어 보이기도 합니다.
- jenkins는 설치할 때 ingress 설정을 미리 할 수 있습니다. 
  그러나 keycloak + phpldapadmin 설치 때 언급했지만 비용절감을 위해 
  같은 namespace 안의 ingress들을 통합해야 하기 때문에 추가적인 일은 필요합니다.

아무튼 설치해 봅시다:
<pre><code>$ helm install jenkins -n mta-cicd -f jenkins-values.yaml stable/jenkins
NAME: jenkins
LAST DEPLOYED: Mon Jul 27 09:03:31 2020
NAMESPACE: mta-cicd
STATUS: deployed
REVISION: 1
NOTES:
1. Get your 'admin' user password by running:
  printf $(kubectl get secret --namespace mta-cicd jenkins -o jsonpath="{.data.jenkins-admin-password}" | base64 --decode);echo
2. Get the Jenkins URL to visit by running these commands in the same shell:
  export POD_NAME=$(kubectl get pods --namespace mta-cicd -l "app.kubernetes.io/component=jenkins-master" -l "app.kubernetes.io/instance=jenkins" -o jsonpath="{.items[0].metadata.name}")
  echo http://127.0.0.1:8080
  kubectl --namespace mta-cicd port-forward $POD_NAME 8080:8080

3. Login with the password from step 1 and the username: admin

4. Use Jenkins Configuration as Code by specifying configScripts in your values.yaml file, see documentation: http:///configuration-as-code and examples: https://github.com/jenkinsci/configuration-as-code-plugin/tree/master/demos

For more information on running Jenkins on Kubernetes, visit:
https://cloud.google.com/solutions/jenkins-on-container-engine
For more information about Jenkins Configuration as Code, visit:
https://jenkins.io/projects/jcasc/
</code></pre>
길게 설명 메시지가 나옵니다. 관리자(admin) 비번을 정하지 않았다면 여기에 비번을 얻는 방법이 기술됩니다.

AWS Load balancer 및 Route53 설정을 끝내면 잠시 후 접속이 가능해집니다. (이건 따로 설명하지 않습니다)

Jenkins에 로그인해서 우선 필요한 설정을 합니다:
- Jenkins 관리 / 시스템 설정 으로 들어가
  - Jenkins URL 확인
  - Global Keycloak Settings / Keycloak JSON
    . keycloak 에 ci realm 을 추가해 두었는데 거기 jenkins 라는 클라이언트를 추가합니다. 
    . 그 추가한 client 설정 중 installation 탭에서 json 을 복사해 둡니다.
    . 이 복사해 둔 json 텍스트를 여기 붙여넣으면 됩니다.
  - Save/Apply
- Jenkins 관리 / Configure Global Security
  - Authentication / Security Realm : Keycloak Authentication Plugin 선택 <-- 이것은 가장 마지막에 합니다. 잘못할 경우 처음부터 작업해야 할 수 있음.
  - Authorization / Strategy : Matrix-based security 선택
    . 이 때 관리자 권한을 부여할 사용자를 지정하는데, 처음에는 위의 Authentication 설정이 잘 되는 지 확신하기 힘드므로 익명사용자도 Admin과 동등하게 
      권한을 주었다가 잘 되는 걸 확인하고 제한을 주는 접근을 하는 게 좋습니다.
  - Save/Apply
여기까지 진행하면 keycloak 인증으로 로그인을 다시 해야 하는데
- 당연히 하나 이상의 사용자가 준비되어 있어야 합니다.
- 하나 이상의 admin 권한의 사용자가 준비되어 있어야 합니다.

## Optional: Jenkins 버전에 따른 변화

어떤 버전의 Jenkins는 위와 같이 설정하고 재시작을 하면 원래대로 되돌아갑니다.

정확히는 설정의 일부가 되돌아갑니다. 그 되돌아가는 설정 중에 Authentication 옵션값이 포함되어 있습니다.
어느 시점부터 2020년 7월 어느날까지의 jenkins는 Configuration as Code 기능을 내장하고 있는데
우습게도 이 설정 일부가 ConfigMap 으로 적용되고 있고 재시작할 때 이 값이 덮어쓰입니다.
그래서 어느 설정은 재시작 해도 변경된 채로 남고 어느 설정은 재시작하면 리셋됩니다.

그러다가 2020-07-28 현재 뭔가 잘못된 걸 알아차렸는지 바꾼 것 같아요.
stable/jenkins 2.1.0 버전에는 Configuration as Code 기능을 포함하지 않아서 재시작해도 괜찮습니다.

만약 문제가 있는 버전을 사용해야 했다면 내가 바꿀 속성이 Configmap에 있을 경우 이를 수정해야 합니다.
아래는 그 수정의 예를 보인 것입니다.
<pre><code>$ kubectl get cm -n jenkins jenkins-jenkins-jcasc-config -o yaml > jenkins-jenkins-jcasc-config-cm.yaml
vi jenkins-jenkins-jcasc-config-cm.yaml 
  jcasc-default-config.yaml: |-
    jenkins:
      authorizationStrategy: legacy
      securityRealm: keycloak
      clouds:
      - kubernetes:
          jenkinsUrl: "https://jenkins.skmta.net"
          jenkinsTunnel: "jenkins-agent.mta-cicd:50000"
    unclassified:
      location:
        url: https://jenkins.skmta.net"
# 내용이 escape character를 포함한 문자열로 되어 있는데 yaml format으로 얻어 표현했음. 바꾸어서 수정해도 되고 문자열을 그대로 수정해도 됩니다.
# 참고로 이걸 yaml 문서 그대로 얻는 방법은 다음과 같습니다:
# kubectl get cm -n mta-infra jenkins-jenkins-jcasc-config -o jsonpath="{.data['jcasc-default-config\.yaml']}"
</code></pre>



