# aws-etude 

이 프로젝트는 제가 속한 조직에서 소그룹으로 진행되는 AWS + Kubernetes + Keycloak(SSO) + CI/CD 구현 중 제가 맡은 일부 작업을 설명하는 곳입니다.

다음 파일들을 포함합니다:

- [instances.md](https://github.com/anabaral/aws-etude/blob/master/instances.md)
  + AWS의 EKS 에서 오토스케일링 그룹을 조정해 사용되지 않을 때 인스턴스를 모두 내리는 스크립트를 소개합니다.
- [ingress_config_and_argocd.md](https://github.com/anabaral/aws-etude/blob/master/ingress_config_and_argocd.md)
  + 같은 네임스페이스의 여러 인그레스를 하나로 통합관리하며 이를 Argo CD로 싱크하는 방법을 소개합니다.
- [jenkins_and_gitea.md](https://github.com/anabaral/aws-etude/blob/master/jenkins_and_gitea.md)
  + 소스 저장소인 Gitea의 설정 (keycloak 연계를 위한 설정, Jenkins 연계를 위한 webhook 설정)
  + Jenkins pipeline의 구성 (Jenkinsfile)
- [jenkins_install.md](https://github.com/anabaral/aws-etude/blob/master/jenkins_install.md)
  + CI 툴의 대표격인 Jenkins의 설치 방법을 기술합니다.
- [keycloak_install.md](https://github.com/anabaral/aws-etude/blob/master/keycloak_install.md)
  + opensource SSO 툴인 Keycloak의 설치 절차를 기술합니다.
