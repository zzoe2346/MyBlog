---
title: Spring MVC 처리 흐름
tags:
  - Spring
date: 2024-08-20
categories: 공부
---

## Spring MVC 처리 흐름

1. **클라이언트 요청**: 클라이언트가 `/products` 엔드포인트에 POST 요청을 보냅니다.
2. **디스패처 서블릿**: 요청을 받아 적절한 핸들러(컨트롤러 메서드)를 찾습니다.
3. **HandlerMapping**: 요청 URL과 컨트롤러 메서드를 매핑합니다.
4. **HandlerAdapter**: 컨트롤러 메서드를 호출하기 위해 필요한 파라미터를 준비합니다. 이 단계에서 `@Valid` 어노테이션이 적용된 파라미터에 대해 유효성 검사를 수행합니다.
5. **유효성 검사**: `Product` 객체의 유효성 검사가 수행되고, 유효하지 않은 경우 `MethodArgumentNotValidException`이 발생합니다.
6. **예외 처리**: 예외가 발생하면 `@ControllerAdvice`로 설정된 전역 예외 처리기가 이 예외를 처리합니다.

## References

- https://docs.spring.io/spring-framework/reference/web/webmvc.html
- https://docs.spring.io/spring-framework/docs/3.2.x/spring-framework-reference/html/mvc.html