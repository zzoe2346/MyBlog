---
title: CustomValidation의 장단점
tags:
  - Java
  - Spring
date: 2024-08-08
---
**장점**

- java코드로 예외처리를 하니 정규표현식을 몰라도 됨
- 애너테이션의 명칭이 명확
- 좀더 정교한 예외처리가 가능

**단점**

- 동료 개발자와 협업하는거라면 만든 사람이 동료에게 직접설명 혹은 주석을 달거나, 동료 개발자가 직접 애너테이션을 분석해봐야하는 단점이 존재
- `@Pattern`같이 그냥 정규표현식과 메시지만 넣으면 끝나는것을 `annotation`, `validator` 까지 만드니 기존의 애너테이션을 사용하는것보다 자원이 들어가는 편
## References
- https://www.baeldung.com/spring-mvc-custom-validator