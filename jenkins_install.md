# jenkins 설치 (keycloak 설정과 함께)

jenkins를 설치하겠습니다. 일전에도 PC 의 virtualbox 기반 k8s 에서 설치한 적이 있는데 비슷하지만 약간 다른 방식으로 진행할 것입니다.

ingress 등 들어가야 하는 항목이 복잡하여 values.yaml 파일을 먼저 작성하겠습니다.
<pre><code>$ vi jenkins-values.yaml
namespaceOverride: mta-cicd
master:
  adminPassword: <정해둔_비번>
  installPlugins: ["credentials-binding:1.23"]
  additionalPlugins: ["keycloak:2.3.0", "gitea:1.2.1", "docker-workflow:1.23"]
  ingress:
    enabled: true
    hostName: jenkins.skmta.net
    annotations: { "kubernetes.io/ingress.class":"alb", "alb.ingress.kubernetes.io/scheme":"internet-facing", "alb.ingress.kubernetes.io/listen-ports":"[{\"HTTP\":80}, {\"HTTPS\":443}]", "alb.ingress.kubernetes.io/target-type":"ip"}
    path: "/*"
persistence:
  storageClass: gp2
</code></pre>
내용을 보자면,
- jenkins 설치할 때 keycloak plugin을 같이 설치하도록 합니다.
  - 이 중 credentials-binding 은 이미 1.22 가 설치되어 있던데.. 1.23으로 업글합니다.
- 관리자 비번을 미리 정합니다. 이 방식은 장단점이 있으므로 좀 생각해 봅시다: 
  - 정하지 않으면 랜덤방식으로 결정되는데 이게 가시성이 안좋아서 불편합니다.
  - 그러나 정하면, kubeapps 같이 설치된 charts 정보를 보여줄 때 이 values.yaml 내용이 평문으로 보입니다. 좀 불안하죠?
  - 허나 keycloak 인증방식을 사용하게 되면 어차피 최초 로그인때만 쓰이는 관리자이므로 큰 문제는 없어 보이기도 합니다.
- jenkins는 설치할 때 ingress 설정을 미리 할 수 있습니다. 어차피 AWS 측 정보를 맞춰 수정해야 하고 
  keycloak + phpldapadmin 설치 때 언급했지만 비용절감을 위해 같은 namespace 안의 ingress들을 통합해야 하기 때문에 일이 없어지는 건 아닙니다..

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
  - Authentication / Security Realm : Keycloak Authentication Plugin 선택
  - Authorization / Strategy : Legacy mode 선택
  - Save/Apply
여기까지 진행하면 keycloak 인증으로 로그인을 다시 해야 하는데
- 당연히 하나 이상의 사용자가 만들어져 있어야 하고
- ci realm 에 admin 이라는 이름의 role을 생성해서 관리자로 쓰일 사용자에게 부여해야 합니다. (legacy mode 에서 관리자를 식별하는 방법)
그것까지 준비되었다면 안심. 그렇지 않다면 jenkins pod 를 죽여 재시작하시면 원래대로 되돌아옵니다.

잠깐, 재시작하면 원래대로 되돌아온다고?
그렇습니다. 위의 설정이 유지가 되지 않습니다. 최신 버전의 jenkins는 Configuration as Code 기능을 내장하고 있는데
우습게도 이 설정 일부가 ConfigMap 으로 적용되고 있어요. 
그래서 어느 설정은 재시작 해도 변경된 채로 남고 어느 설정은 재시작하면 리셋됩니다.
뭔가 잘못하고 있는 느낌인데 2020년 7월 현재까지의 버전은 그러합니다.
이걸 영속화 하기 위해 Configmap 을 수정합시다:
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

여기까지 하면 재시작 해도 keycloak 인증을 계속 요구할 겁니다.


아직 아쉬운 것은 권한관리 방식인 legacy 입니다. 이 방식은 심플하게
- admin 권한이 부여되면 관리자
- 그냥 로그인 사용자는 자기 프로젝트 관리 가능
- 익명 사용자는 보는 건 가능
인데 제일 마지막 익명 사용자 통제가 마음대로 안됩니다. 결국 matrix autorization 을 도입해야 할 거 같은데 현재는 보류...


