---
layout: post
title:  "AWS LAMBDA로 회원가입 구현 : API GATEWAY and CORS"
---
구글 IO 행사 때 No-Ops에 대한 강연을 들은 후에 AWS Lambda로 서비스를 만들어보고 싶었다. 써보려고 둘러봤는데 그 때는 함수를 하나하나 올려야되는 줄 알고 불편해서 쓰기를 포기했었다.

그러다 얼마 전 친구가 [Serverless](https://github.com/serverless/serverless)라는 프레임워크를 소개해줘서 사용해보기로 마음을 먹었다. [이걸](https://github.com/pmuens/serverless-crud) 이용해서 기본적인 DynamoDB와 Lambda를 이용한 기본적인 CRUD를 만들었는데, 로그인 구현이 문제였다. 암호화 등은 설명이 잘 되어있어서 구현하는데 큰 어려움을 못 느꼈는데, 세션 유지가 문제였다.

JWT Token을 쿠키에 저장하고 이를 이용해서 세션을 유지하는 방식을 사용하려고 했는데, Cross Origin에 관련한 에러가 자꾸 떴다.

# Serverless 프레임워크는 AWS APIGateway를 사용한다.

> API 문서를 정독하자

`serverless.yml` 파일에 보면 각 Lambda 함수로 접근하기 위한 http path를 설정하는 부분이 있다. 이 path는 Lambda 서비스 안에서 설정되는 것이 아니라, APIGateway라는 서비스에서 생성되고 이 서비스가 이를 Lambda로 연결시켜준다. 또한 기본적으로 lambda-proxy integration을 사용한다. APIGateway에서는 Request를 받고 Response를 전송하는 것까지 처리할 수 있는데, lambda-proxy integration을 사용하면 이 request를 그대로 Lambda의 event로 넣어주고 response도 Lambda의 `context.succeed(res)` or `context.done(err, res)` or `context.fail(err)`를 통해 전달된 값이 그대로 넘어간다.

세션 쿠키를 설정하기 위해 `Set-Cookie` 헤더를 설정하려는 도중 APIGateway에서 lambda-proxy integration을 해제하고 Response를 APIGateway를 통해 주어야 한다는 글을 봤다. 그래서 `serverless.yml`에서 integration을 lambda로 설정했더니(lambda-proxy가 기본값이다.), Lambda 함수로 들어오는 event의 형태가 바뀌었고(ex. lambda-proxy일 때는 `event.body`가 JSON으로 들어와 `JSON.parse(event.body)`가 필요하지만, lambda일 때는 이미 파싱된 Object 형태로 들어온다. ), 내가 보낸 Response가 그대로 처리되지 않았다.

<pre>

{ statusCode: 200, 

headers: { "Access-Control-Allow-Origin" : " *",

"Access-Control-Allow-Credentials": true },

body: { "message": "success" }

}

</pre>

위와 같은 Response를 주었을 때 headers 부분이 무시되고 APIGateway에서 설정해준 헤더만이 나타났다.

원래 Set-Cookie 헤더를 주는 법을 찾다가 integration을 lambda로 주는 위의 방법을 시도한건데, 사실은 integration은 lambda-proxy로 그대로 두고 headers에 Set-Cookie라는 걸 넣어주면 되는 거였다.



# Access-Control-Allow-Credentials가 true일 때 Access-Control-Allow-Origin은 *(wildcard)일 수 없다.

> 진짜인가? 되는 걸 보이긴 쉬워도 아닌 걸 확인하긴 어렵다.

다음 문제는 이거였다. 리퀘스트를 보낼 때마다 쿠키를 보내고, 로그인 등에서는 Response의 Set-Cookie 헤더를 받아다가 쿠키를 써야하니까 axios의 withCredentials 옵션을 true로 줘야 한다. (이 옵션은 쿠키를 보내는 것에만 해당하지 않고 Set-Cookie 헤더를 읽어다가 쿠키를 쓰는 것에도 해당한다. / jQuery나 다른 ajax 라이브러리에도 비슷한 옵션이 있을 것이다.)  그러나 이게 켜져있을 경우에는 Access-Control-Allow-Origin이 반드시 Request를 보낸 URL이어야 한다. 처음에는 localhost:3000은 서버 입장에서 스스로의 3000 포트가 될 테니까 내 로컬 컴퓨터는 안될 거고, 내 로컬 컴퓨터의 IP를 알아내서 써야한다고 생각했다. 그러나 그냥 http://localhost.3000이라고 쓰면 되더라.

그리고 답답했던 건 Access-Control-Allow-Origin은 하나의 주소만 가능하고 배열 등 여러 주소를 넣을 수는 없었다. (가능하다면 알려주세요.) 람다 함수 안에 WhiteList 배열을 만들고 만약 event.Origin이 해당 배열에 포함되어 있다면 `Access-Control-Allow-Origin: (해당 주소)`로 주는 방법으로 우회할 수는 있다. 이 제약조건이 왜 있는건 지 궁금하다. (withCredentials 켤 경우 와일드카드 안되는 것, 그리고 배열로 줄 수 없는 것)



# 쿠키는 클라이언트 사이드 주소가 아니라 서버 사이드 주소에 저장된다.

> 컴퓨터가 없는 걸 가지고 일을 할 리가 없다.

회원가입을 하면 쿠키 설정이 잘 됐는데, 로그인을 하는 경우에는 쿠키 설정이 안 됐다. 그리고 로그아웃도 안됐다. 그래서 `localhost:3000` (내가 클라이언트 코드를 돌리고 있는 주소)에 해당하는 쿠키를 삭제해봤다. 그런데 리퀘스트에 쿠키가 그대로 살아있으면서 로그인이 유지되는 것이다! 이게 뭔가 하면서 쿠키를 열심히 뒤져보다가 크롬 브라우저 주소창 왼쪽의 (자물쇠 Secure)나 ( 동그라미 안에 i )를 눌러보면 현재 사이트의 쿠키 정보가 나온다는 사실을 발견했다. (이런 걸 쉽게 쓰지만 이거 하나 찾는데 몇 시간씩 걸렸다.) 그런데 쿠키가 `localhost:3000`이 아니라` my-api-server.com/users`에 설정되어 있었다. 이 쿠키를 삭제하니 로그인이 해제되었다.

그 다음으로 발견한 현상은 현재 로그인된 유저를 가져오는 request에서 쿠키를 두 개나 보내고 있는 것이었다. 회원가입을 하는 함수는 [POST] `my-api-server.com/users`였고 로그인을 하는 함수는 [POST] `my-api-server.com/users/token`이었다. 따라서 하나의 쿠키는 `/users`에, 하나의 쿠키는 `/users/token`에 설정되었고 이 두 개의 쿠키가 중복되어 설정되어 있어서 [GET] `my-api-server.com/users`로 현재 로그인한 유저 정보를 가져올 때 `/users`의 하위에 있는 모든 쿠키가 전송되고 있었다. 로그아웃 ([DELETE] `my-api-server.com/users/token`)을 하더라도 `/users`에 설정된 쿠키는 삭제되지 않았고, 따라서 마지막으로 가입한 회원의 쿠키는 그대로 유지되었던 것이다. 이는 쿠키를 설정할 때 `PATH=/`을 설정해줌으로써 해결하였다.







사소해 보이는 걸로 이틀이나 고생을 해서 시간이 너무 아까웠다. 지금 이걸 쓰는 것까지 하면 이틀하고 반 정도는 되는 것 같다. 그래도 비교적 뻔한 작업을 반복하던 와중에 새로운 걸 배우는 기회였다. 네트워크나 보안에 관한 수업도 들어보고 싶다.