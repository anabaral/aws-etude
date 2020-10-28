# Wordpress 와 keycloak 연동

Wordpress의 설치과정은 [Google cloud 실습](https://github.com/anabaral/gcloud-etude)에서 다룬 적 있기 때문에 
상세설명은 생략.  
그리고 그때와 달리 여기서는 샘플 어플리케이션 하나 설치해 연결하는 의미만 있으므로 더더욱 간략히 적겠음.

```
$ cat wordpress-values.yaml
wordpressUsername: "wordpress"
wordpressPassword: "정해둔비번"
wordpressBlogName: "Wordpress_Sample_Blog"
wordpressFirstName: "Bill"
wordpressLastName: "Gates"
wordpressEmail: "anabaral@xmail.com"
persistence:
  existingClaim: wordpress-pvc  # 재설치를 많이 하려다 보니 미리 만들어두었음.
metrics:
  enabled: true
replicaCount: 1
service:
  type: ClusterIP
```
설치하면 다음과 같이 wordpress와 mariadb가 하나씩 뜸.
```
$ kubectl get po -n mta-dev
...
wordpress-7c47bfdbcd-svw46        2/2     Running     0          16h
wordpress-mariadb-0               1/1     Running     0          16h
```

관리자(위의 파일에서 정한 이름과 비번)로 접속해 Woocommerce 등의 플러그인들을 수동설치하고 한 플러그인을 찾아 추가설치하자:
- miniorange-openid-connect-client
이것 말고도 OIDC Client 가 한두 개 더 보이던데 비슷하리라 생각된다.


그리고 나면 좌측 메뉴판에 새로운 메뉴가 등장한다:  
![메뉴](https://github.com/anabaral/aws-etude/blob/master/img/wordpress-oidc-screen1.png)  
이것을 들어가서 셋업하면 된다. 위 그림에는 이미 하나 셋업되어 있고, Premium이 아니면 하나만 셋업할 수 있다..  
아래 그림은 편집 화면인데, 신규 화면도 아주 다르지는 않다. (다만, 신규 화면에서 정한 Display App Name을 편집 화면에선 다시 바꾸지 못함)

![편집화면](https://github.com/anabaral/aws-etude/blob/master/img/wordpress-oidc-screen2.png)  

각 항목들은 다음과 같다:
- Redirect URL : SSO 로그인 성공 후 들어가는 페이지 URL
- Client ID : 클라이언트의 아이디. 여기서는 Keycloak에서 dev 렐름에 미리 클라이언트를 만들어 두었고 만들 때 wordpress-oidc 라 정했다. 
- Client Secret : Keycloak에서 정한 시크릿. Keycloak 에서 클라이언트 Access Type에 Confidential 을 지정하고 Credential 탭에서 얻는다.
- Scope : openid 라고 기입
- Authorization Endpoint : https://keycloak.skmta.net/auth/realms/dev/protocol/openid-connect/auth     # dev 가 realm name
- Access Token Endpoint : https://keycloak.skmta.net/auth/realms/dev/protocol/openid-connect/token     # dev 가 realm name
- login button : [v] Show on login page

Test Configuration 이 성공하면 그대로 저장하면 끝.

선택사항일 수도 있지만 기본으로 저장하면 Gitea나 Jenkins와는 달리 keycloak 사이트로 리다이렉트 하지 않는다.  
그냥 로그인 버튼만 다른 걸 누르면 끝.

![로그인화면](https://github.com/anabaral/aws-etude/blob/master/img/wordpress-login-screen.png)  
화살표로 표시한 버튼이 OIDC 로그인 버튼이다. (그냥 Wordpress로 로그인한다(?)고 나오는데.. 그래서 Display App Name 을 처음에 줄 때 잘 줘야 함)

