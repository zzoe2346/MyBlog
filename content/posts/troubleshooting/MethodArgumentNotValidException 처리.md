---
title: MethodArgumentNotValidException 처리
tags:
  - Spring
date: 2024-08-17
---
## 💡문제 인지
`MethodArgumentNotValidException`가 발생하는데 Controller 층에서 catch가 안되었다.  그런데 이걸 catch하려고 Controller Class의 이를 처리하는 Method에 아무리 try-catch를해도 예외가 안잡혔다. 

## 🤔원인
`MethodArgumentNotValidException` 는 Spring MVC에서 Controller에 전달되기전 VIEW,MODEL 에서 예외가 발생해서 컨트롤러 안에서는 catch가 불가했던것임. 
## 🎉해결

`@ControllerAdvice`는 전역 예외 처리가 가능 + 컨트롤러 예외처리 가능!
`@ExceptionHandler(MethodArgumentNotValidException.class)`로 `MethodArgumentNotValidException`을 catch해서 `"version-SSR/product-error"` 페이지로 리턴해줌
## 예제 코드

컨트롤러와 예외 처리기를 포함한 예제 코드를 다시 확인해 보겠습니다.

### 컨트롤러 클래스

```java
@RestController
public class ProductController {

    @PostMapping("/products")
    public ResponseEntity<Product> createProduct(@Valid @RequestBody Product product) {
        // Product 객체가 유효한 경우 비즈니스 로직 수행
        return ResponseEntity.ok(product);
    }
}
```

### 전역 예외 처리기

```java
@ControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(MethodArgumentNotValidException.class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    public String handleValidationExceptions(MethodArgumentNotValidException ex) {
        return "version-SSR/product-error";
    }
}
```
## Reference
- https://docs.spring.io/spring-framework/docs/3.2.x/spring-framework-reference/html/mvc.html
- https://docs.spring.io/spring-framework/reference/web/webmvc.html
- [[Spring MVC 처리 흐름]]