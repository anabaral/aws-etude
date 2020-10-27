# aws-etude 

이 프로젝트는 제가 속한 조직에서 소그룹으로 진행되는 AWS + Kubernetes + Keycloak(SSO) + CI/CD 구현 중 제가 맡은 일부 작업을 설명하는 곳입니다.

다음 파일들을 포함합니다:

- [instances.md](https://github.com/anabaral/aws-etude/blob/master/instances.md)
  + AWS의 EKS 에서 오토스케일링 그룹을 조정해 사용되지 않을 때 인스턴스를 모두 내리는 스크립트를 소개합니다.
- [ingress_config_and_argocd.md](https://github.com/anabaral/aws-etude/blob/master/ingress_config_and_argocd.md)
  + 같은 네임스페이스의 여러 인그레스를 하나로 통합관리하며 이를 Argo CD로 싱크하는 방법을 소개합니다.
- [jenkins_and_gitea.md](https://github.com/anabaral/aws-etude/blob/master/jenkins_and_gitea.md)
  + 소스 저장소인 Gitea의 설정 (keycloak 연계를 위한 설정, Jenkins 연계를 위한 webhook 설정)
  + 샘플 어플리케이션 js-console 등록 및 배포 준비
  + Jenkins pipeline의 구성 (Jenkinsfile)
- [jenkins_install.md](https://github.com/anabaral/aws-etude/blob/master/jenkins_install.md)
  + CI 툴의 대표격인 Jenkins의 설치 방법을 기술합니다.
- [keycloak_install.md](https://github.com/anabaral/aws-etude/blob/master/keycloak_install.md)
  + opensource SSO 툴인 Keycloak의 설치 절차를 기술합니다.
- [install_elasticsearch.md](https://github.com/anabaral/aws-etude/blob/master/install_elasticsearch.md)
  + elasticsearch 시험설치 결과를 기록하였습니다. 2020-10 현재 떠 있는 Elasticsearch는 이 문서와 무관합니다.
- Nodejs 로 MTA-admin 개발하기
  + 관리 어플리케이션을 Nodejs로 개발하면서 이것저것 시도하거나 배웠던 것들을 기록하였습니다.
  + [nodejs-etude.md](https://github.com/anabaral/aws-etude/blob/master/nodejs-etude.md)
    * 처음 개발 시작하면서 알게 되거나 시도했던 것들을 적었습니다.
  + [nodejs-db.md](https://github.com/anabaral/aws-etude/blob/master/nodejs-db.md)
    * 관리 대상 테이블을 기술합니다.
  + [nodejs-dev-keycloak.md](https://github.com/anabaral/aws-etude/blob/master/nodejs-dev-keycloak.md)
    * 개발 어플리케이션은 keycloak 인증을 거쳐야 접근 가능하도록 하였는데 이 때 겪은 것들을 기술합니다.
  + [nodejs-dev.md](https://github.com/anabaral/aws-etude/blob/master/nodejs-dev.md)
    * 어플리케이션 개발 관련 이야기들을 적었습니다.
- [troubleshootings.md](https://github.com/anabaral/aws-etude/blob/master/troubleshootings.md)
  + 다른 문서에서 다루기 애매하지만 많이 고생했던 트러블과 그 해결기를 다룹니다.
