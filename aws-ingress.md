# AWS에서 Ingress 설정

예전에 다른데 적었다고 생각했는데 안보여서 다시 정리합니다.

최종 정리는 나중에 하기로 하고 일단 아는 걸 최대한 부어넣겠습니다...

AWS의 Ingress 특징을 살펴보면 다음과 같습니다:
* 일단 설정하고 나면 k8s의 Ingress 설정을 바꿀때마다 ALB의 설정이 동기화 됩니다.
  - 이게 정말 골치아픈게, 웹 콘솔에서 열심히 설정한 게 (Ingress 쪽으로는 역 동기화가 안되면서) Ingress를 조금 건드리면 리셋되어 버립니다.
* 물론 Ingress를 만든다고 바로 ALB에 연결되는 것은 아닙니다. 자세한 건 생성 가이드에서..



## ALB 생성

나중에


## Ingress 설정 

Ingress 는 k8s 외부와의 인터페이스이므로 맨땅에서 k8s를 조립하는 특이한 케이스를 제외하곤 CSP와 긴밀한 관계를 가집니다.  
그래서 Annotation이 정보 이상의 매우 중요한 의미를 가지곤 합니다.

이를테면 다음은 Ingress 지원하는 주체를 AWS의 ALB로 사용하겠다는 약속입니다.
```
metadata:
  annotations:
    kubernetes.io/ingress.class: alb
```
예를 들어 kubernetes 제공 ingress-controller 를 사용할 경우에는 이 값을 `nginx` 로 씁니다.

AWS 관련해선 좀 찾아보니 참조할 링크가 존재하네요 https://sarc.io/index.php/aws/2084-eks-ingress-https

다른 설정은 다음과 같이 합니다:
```
    ## 80 요청이 올 경우 443으로 리다이렉트 하라는 설정. 이걸 ALB 에 직접 설정할 수도 있지만 Ingress 변경시 리셋됨
    alb.ingress.kubernetes.io/actions.ssl-redirect: '{"Type": "redirect", "RedirectConfig":
      { "Protocol": "HTTPS", "Port": "443", "StatusCode": "HTTP_301"}}'
    ## AWS Cert-Manager 의 인증서를 SSL용으로 사용할 경우 아래와 같이 ARN을 지정
    alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:ap-northeast-2:046492796127:certificate/175b73ca-de62-41e8-82ae-38522003d348
    ## 외부 접속용 LB를 붙이는 거니 'internet-facing' 입니다.
    alb.ingress.kubernetes.io/scheme: internet-facing
    ## target-type은 ingress 사용하면 'ip' 입니다. service에 NodePort 방식으로 연결하려면 'instance'를 쓴다네요
    alb.ingress.kubernetes.io/target-type: ip
```

주의할 사항이 하나 있는데, 
```
  - host: jenkins.polaris.net
    http:
      paths:
      - backend:
          service:
            name: jenkins
            port:
              number: 8080
        path: /*               # <==== 와일드카드를 반드시 명시하지 않으면 '/' 만 받아들임
        pathType: ImplementationSpecific
```
위와 같이 AWS ALB용 Ingress에서는 와일드카드를 반드시 써야 합니다. (kubernetes ingress-controller 는 그냥 '/' 만 써도 됩니다)

