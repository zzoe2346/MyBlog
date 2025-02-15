---
title: preflight request관련 CORS 에러
tags:
  - HTTP
date: 2024-07-30
categories: 트러블슈팅
---

## 문제 정의

기존의 설정으로 헤더를 이용하는 위시관련기능에서 계속 cors error가 발생하였습니다.

## 문제 분석

- 브라우저의 에러로그를 살핌
- 검색

```
Access to XMLHttpRequest at 'http://43.201.254.198:8080/api/wishes?page=0&size=10&sort=createdDate,desc' 
from origin 'http://localhost:3000' has been blocked by CORS policy: Response to preflight request doesn't pass
 access control check: It does not have HTTP ok status.
```

## 🤔원인

여기서 생소한 preflight request를 보았고 아래자료와 GPT 등으로 무엇인지 알아보았습니다.  
[CORS관련 문서](https://developer.mozilla.org/ko/docs/Web/HTTP/CORS#http_%EC%9A%94%EC%B2%AD_%ED%97%A4%EB%8D%94)

## 문제 해결 과정

위시 관련 api 요청에는 Authourization 헤더를 요구합니다. 이 요청은 단순한(GET, POST 등)요청으로 취급되지 않고 복잡한 요청으로 브라우저에서 취급을 합니다. 그래서 CORS 정책에서
`preflight request`이 요청은 서버가 실제 데이터 요청을 허용할지 여부를 확인하는 과정입니다.`preflight request`가 먼저 요청이 왔을때 아래의 응답을 원하고 있었습니다.

```httpspec
HTTP/1.1 200 OK
Access-Control-Allow-Credentials: true
```

## 개선 결과

그런데! 현재 서버는 위시기능에 들어오는 요청이 컨틀롤러에 오기 전에 인터셉터로 낚아서 헤더에 토큰이 있냐 없냐로 단순한 처리만 하고있었던 것입니다. 그래서 위의 에러가 요청하는것을 인터셉터의`preHandle`에
넣어주니 성공적으로 위시 기능을 활용할 수 있었습니다.

```
@Override
public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) {
        if(request.getMethod().equals("OPTIONS")){
            // OPTIONS 요청에 대해 상태 코드 200을 반환
            response.setStatus(HttpServletResponse.SC_OK);
            // 자격 증명 허용
            response.setHeader("Access-Control-Allow-Credentials", "true");
            return false;
        }
```