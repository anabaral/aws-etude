# nodejs 개발에서의 keycloak 연동

## frontend

## backend 

### keycloak-connect 버그

확실히 버그인지, 아니면 내가 뭔가 잘못한 건지는 알 수 없지만 이런 문제가 있다:
```
Chrome 콘솔에서 뜨는 메시지:
Access to XMLHttpRequest at 'https://keycloak.skmta.net/auth/realms/dev/protocol/openid-connect/auth?client_id=mta-manage-log&state=70e9c859-a13f-4a4a-8947-5d1c5674d0df&redirect_uri=http%3A%2F%2Fmynode-selee.skmta.net%2Fapi%2Falert_summary%2Fget%3Ffrom_dt%3D2020102000%26auth_callback%3D1&scope=openid&response_type=code' (redirected from 'https://mynode-selee.skmta.net/api/alert_summary/get?from_dt=2020102000') from origin 'https://mynode-selee.skmta.net' has been blocked by CORS policy: Response to preflight request doesn't pass access control check: No 'Access-Control-Allow-Origin' header is present on the requested resource.
```
처음에는 그냥 일반적인 CORS 문제라고 보고 keycloak의 CORS(Web Origin) 설정과 nodejs 에서의 설정을 찾아보았지만
결국 전혀 다른 곳에서 원인을 발견해 버렸다.

https://github.com/keycloak/keycloak-nodejs-connect/blob/master/middleware/check-sso.js 이나<br>
https://github.com/keycloak/keycloak-nodejs-connect/blob/master/middleware/protect.js
안을 보면 
```
  let protocol = request.protocol;
  ...
  let redirectUrl = protocol + '://' + host + (port === '' ? '' : ':' + port) + (request.originalUrl || request.url) + (hasQuery ? '&' : '?') + 'auth_callback=1';
```
같은 구절이 있는데, 이게 서버사이드 요청에 대한 리다이렉트를 결정한다.

그런데 리다이렉트를 하면 무조건 protocol이 http 로 정해져 버린다.
왜냐하면 다음 구조 때문.
```
[https_req] ----> ingress / L4 ----> [http_req] ----> real_app
```
그러면 위의 코드는 ```real_app``` 에게 전달되는 요청만을 분석하기 때문에 https 값이 올 수 없고,
브라우저는 http로 시작하는 redirect uri 를 받게 된다.

즉 redirect 시키는 구조는 어차피 안됨.

bearer-only 모드로 다시 도전해 봐야겠다.




