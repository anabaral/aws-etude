# nodejs 개발에서의 keycloak 연동

## frontend

```keycloak-init.js``` 파일을 작성한다:
```
  var login = {};
  var kc = Keycloak();
  var initOptions = {
        onLoad: 'login-required'
      };
  kc.init(initOptions).then(function(authenticated) {
      if (!authenticated) {
          alert('Not authenticated');
      } else {
          kc.loadUserProfile();
          $(document).ready( function(){
            $(document.body).css("display","block");
          });
      }
    }).catch(function() {
        alert('Init Error');
    });
```
이 파일을 html 파일에 포함시키면 되는데, 문제는 이 로직의 실행이 document가 다 읽히기 전에 실행되어야 하지만 한편으로는 body 엘리먼트가 어느 정도 구성된 상태에서 실행되어야 하기도 하므로 (내부 로직에서 body 엘리먼트 안에 iframe을 삽입하더라) 보통 body의 끝 부분에 넣는다:
```
<html>
<head>...</head>
<body>
...
<script type="text/javascript" src="/js/keycloak-init.js"></script>
</body>
</html>
```

이렇게 하면 html 자체에 대한 보안 처리가 된다. html 로드하는 시점에 미인증 상태이면 keycloak 사이트로 로그인하러 가며(=redirect),
로그인 하고 나면 사용자 정보를 얻어오게 된다.

여기서 조금 문제가 있는데, 나는 document.onLoad 시점에 서버로부터 정보를 얻어 초기화면을 구성하게 했는데, 이게 두 번 호출된다.
* 미인증 상태에서의 첫 호출은 당연히 redirect하니까 결과가 있든 없든 무시되고 두번째 호출이 결과를 얻는다.
* 그러나 인증하고 나서도 매번 keycloak 으로 인증했음을 확인하기 때문에, 인증 후에 화면 깜박임이 생긴다.
* 위 문제는 일단 body를 안보이게 하고 위의 keycloak-init.js 에서 ```$(document.body).css("display","block");``` 코드로 
  두번째 호출 결과만 보이게 했다. 이건 편법이긴 한데...
* 이 편법을 쓰고 나서도 여전히 두 번 호출한다는 문제는 남아있다. 아마 document.load 시점에 구성하는 초기화면을 
  document.load 조건과 인증 조건을 같이 충족할 때 구성하도록 고쳐야 제대로 동작할 것 같지만 두 동작이 다 비동기적으로 이루어지므로
  아주 깔끔하게 처리하기가 쉽지 않다. 이건 시간 관계상 완전히 해결하지는 못했다.

일단 화면은 인증만 되면 들어갈 수 있게 만들어 두었다. 나중엔 권한 체크도 필요하겠지..

화면을 구성하는 방식은 다음 웹 개발 방식들 중 두 번째를 택했는데
![웹개발](https://github.com/anabaral/aws-etude/blob/master/img/web_dev_diagram.svg)
이 방식에서는 서버에 Ajax를 통해 주요 정보를 받아오게 된다.
당연히 앞서 구현한 화면 보안 뿐 아니라 서버 사이드 보안도 필요하다. 이건 화면단에서의 구현도 필요한데 backend에서 같이 다루겠다.

## backend 

아래 사이트를 참조했는데.. 실제 설명이 아주 충분하지는 않아서 애 좀 먹었다:
https://www.keycloak.org/docs/latest/securing_apps/#_nodejs_adapter

일단 
```
var express = require('express');
var session = require('express-session');
var Keycloak = require("keycloak-connect"); // SSO
var memoryStore = new session.MemoryStore();
var kcConfig = JSON.parse(fs.readFileSync('public/keycloak.json')); // 이렇게 주지 않으면 server.js 와 같은 위치에 파일이 있어야 해서 파일 두개 필요
kcConfig['bearerOnly'] = 'true';                                    // 이건 꼭 필요한 건지 불확실
var keycloak = new Keycloak({ store: memoryStore }, kcConfig);
...
app.use(session({
  secret: 'mySecret',
  resave: false,
  saveUninitialized: true,
  store: memoryStore
}));
app.use( keycloak.middleware() ); // 이 라인이 app.use(session({...})); 라인보다 뒤에 있어야 함

// 모듈로 분기할 때 패턴
require('./app/alert-routes.js')(app,keycloak); // keycloak을 같이 보내야 모듈에서 사용
// 바로 호출할 때, 혹은 모듈에서 호출할 때 패턴
module.exports = function (app, keycloak) { // 모듈
...
  app.get('/api/alert/get', keycloak.protect(), function (req, res) {
      getAlerts(req, res);
  });
};
```
위와 같이 ```keycloak.protect()``` 만 추가해 주면 인증되었을 때 실행되고 미인증 상태일 때는 403 에러를 낸다.

이 코드가 서버단에 있으면 클라이언트 단에서는 어떻게 코딩해 줘야 하냐면 다음과 같다:
```
// 앞뒤 다 빼고 ajax 구문만 보이면
    $.ajax({
        url: "/api/alert/get",
        method: "GET",
        data: { 'pageSize': pageSize, 'pageNo' : page, ...... },
        headers: { "authorization": "bearer " + kc.token },       // 이게 인증을 위한 핵심
        dataType: "html",
        success: function(response){   ...생략...   }
   })
```
현재는 실패처리를 하지 않고 그냥 멍청하게 끝나게 했는데, alert라도 내려고 보니 앞서 언급한 두 번 호출의 문제가 있어 그렇게는 못하겠다.
뭔가 근원적인 해결이 필요할 것 같은데 시간이 없네... 

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




