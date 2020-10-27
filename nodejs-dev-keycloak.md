# nodejs 개발에서의 keycloak 연동

## frontend

전체적인 화면을 구성하는 방식은 다음 웹 개발 방식들 중 두 번째를 택했는데
![웹개발](https://github.com/anabaral/aws-etude/blob/master/img/web_dev_diagram.svg)
이 방식에서는 서버에 Ajax를 통해 주요 정보를 받아오게 된다.

```keycloak-init.js``` 파일을 작성한다:
```
  var login = {};
  var kc = Keycloak();
  var initOptions = {
        onLoad: 'login-required'
      };
  const kcPromise = new Promise(function(resolve, reject) {
    kc.init(initOptions).then(function(authenticated) {
      if (!authenticated) {
          alert('Not authenticated');
          reject('not authenticated');
      } else {
          kc.loadUserProfile();
          resolve('authenticated');
          $(document).ready( function(){
            $(document.body).css("display","block");
          });
      }
    }).catch(function() {
        alert('Init Error');
    });
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
로그인 하고 나면 사용자 정보를 얻어오게 된다. 다음 그림과 같이:
![HTML에서의 인증](https://github.com/anabaral/aws-etude/blob/master/img/keycloak_auth_html.svg)


여기 코드의 특징을 좀 살펴보면 다음과 같다:
* 인증을 얻은 후 작업과 document.load 후의 화면구성(render) 작업은 둘 다 callback 형태로 실행되는데,
  성격상 인증과 document.load 가 다 완료된 후에 실행되어야 하는 작업이 있다. 서버에 요청해서 화면 구성에 필요한 데이터를 받는 작업이다.
* 이걸 Promise 를 이용해 해결했다. 이걸 쓰지 않으면 
  - 서버 요청 작업을 두 번 하고 첫 번째 작업은 버리거나
  - 아예 인증을 얻지 못한 상태에서 서버에 요청을 하게 될 수도 있다.
* 이것과는 별개로, 인증확인을 위해 keycloak에 호출하는 단계가 강제로(=인증 된 후에도) 들어가다 보니 
  html 호출도 두 번 하게 되고 결과적으로 화면 깜박임이 발생했다.
  - 이걸 해결해 주기 위해 HTML엔 ```<body style="display:none" > ``` 를 두어 처음엔 화면을 보이지 않다가
    document.load 후에 ```$(document.body).css("display","block");``` 실행하는 식으로 넘겼다.
  - 아마 제대로 하려면 인증확인을 redirect 없이 하고 미인증 시에만 redirect하는 방식을 도입해야겠다.
    현재는 그것이 그것대로 연구가 필요한데 시간이 부족해서 통과.

참고로 위에서 언급한 Promise 처리는 다음과 같이 했다:
```
  $(document).ready(function(){
    kcPromise.then(function (){
      $("#menu").load("/menu.html");
      ...
      render_XXXX(); // 서버에 요청해 데이터 받고 화면 그리는 로직 실행
    }).catch(function(err){
      console.log(err);
    });
  });
```

현재 권한 부여 및 체크는 없어서 인증만 되면 화면에 들어가 모든 작업을 실행할 수 있게 만들어 두었다. 나중엔 권한 체크도 필요하겠지..

그런데 위의 다이어그램을 보면 데이터 요청할 때의 인증, 즉 서버 사이드 인증은 빠져 있다. 이는 다음 backend에서 다룬다.

## backend 

아래 사이트를 참조했는데.. 실제 설명이 아주 충분하지는 않아서 애 좀 먹었다:
https://www.keycloak.org/docs/latest/securing_apps/#_nodejs_adapter

아래 소스는 서버사이드 nodejs 스크립트 일부이다:
```
var express = require('express');
var session = require('express-session');
var Keycloak = require("keycloak-connect"); // SSO
var memoryStore = new session.MemoryStore();
var kcConfig = JSON.parse(fs.readFileSync('public/keycloak.json')); // 이렇게 주지 않으면 server.js 와 같은 위치에 파일이 있어야 해서 파일 두개 필요
kcConfig['bearerOnly'] = 'true';                                    // 이건 꼭 필요한 건지 불확실했는데, 필요한 것 같음.
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

앞의 backend 인증에서 ```kcConfig['bearerOnly'] = 'true';  ``` 코드가 필요한 것 같다고 했는데 그 이유를 기술한다.
이 코드가 없으면서 미인증 상태인 경우 아마도 로그인 화면으로 보내는 모양인데 그 과정에서 아래와 같은 오류가 뜬다.
```
Chrome 콘솔에서 뜨는 메시지:
Access to XMLHttpRequest at 'https://keycloak.skmta.net/auth/realms/dev/protocol/openid-connect/auth?client_id=mta-manage-log&state=70e9c859-a13f-4a4a-8947-5d1c5674d0df&redirect_uri=http%3A%2F%2Fmynode-selee.skmta.net%2Fapi%2Falert_summary%2Fget%3Ffrom_dt%3D2020102000%26auth_callback%3D1&scope=openid&response_type=code' (redirected from 'https://mynode-selee.skmta.net/api/alert_summary/get?from_dt=2020102000') from origin 'https://mynode-selee.skmta.net' has been blocked by CORS policy: Response to preflight request doesn't pass access control check: No 'Access-Control-Allow-Origin' header is present on the requested resource.
```
처음에는 그냥 일반적인 CORS 문제라고 보고 keycloak의 CORS(Web Origin) 설정과 nodejs 에서의 설정을 찾아보았지만 별 이상이 없었고
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

그런데 리다이렉트를 하면 무조건 protocol이 http 로 정해져 버린다. 위의 Chrome 콘솔을 잘 보면 redirect_uri 값이 그러하다.  
왜냐하면 다음 구조 때문.
```
※ 현재 호출의 구조는 외부에서 https redirect까지 해서 https를 강제하지만 L4 안쪽은 http 통신을 하게 됨. 즉
[https_req] ----> ingress / L4 ----> [http_req] ----> real_app
```
그러면 위의 코드는 ```real_app``` 에게 전달되는 요청만을 분석하기 때문에 https 값이 올 수 없고,  
브라우저는 http로 시작하는 redirect uri 를 받게 된다.

즉 redirect 시키는 구조는 현재로선 아예 쓰지 않아야 할 것 같다.

