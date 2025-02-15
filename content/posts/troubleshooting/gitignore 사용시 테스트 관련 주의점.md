---
title: .gitignore 사용시 테스트 관련 주의점
tags:
  - Git
date: 2024-08-17
---
## 문제 정의
멘토님께 리뷰를 받았는데 실패하는 테스트가 많다는 리뷰를 받았습니다. 분명 테스트가 모두 통과되는 것을 확인하고 리뷰요청을 했는데 왜 멘토님께서는 테스트가 실패된것일까?
![[../../images/스크린샷 2024-08-09 16.14.44.png]]
## 문제 분석
이번에 `.gitignore`을 처음으로 설정해두었는데 이것이 원인이었습니다. `application-dev.properties`에 전에는 클래스 파일안에 상수화 해둔 카카오 API URI, Redirect URI등을 `@Value`로 지정해서 사용하기위해 저장해 두었는데요. 이번에 과제를 진행하며 kakao client Id같은 민감한 정보는 숨겨놓으라는 요구사항이 있어 이를 위해 .gitignroe에 추가하였었습니다. 

이것덕에 깃허브에 보안상 중요한 코드들이 숨겨졌지만 하지만, 이`application-dev.properties`가 있어야 필수적인 값들을 알고 Bean도 생성하고 하는데 이것이 Github에 푸시가 되면서 `application-dev.properties`가 없어지는 바람에 테스트가 엄청 실패하는 결과가 발생한것으로 원인 파악이 되었습니다.

## 해결 과정
인터넷에서 어떻게하면 해결할 수 있을지 자료를 찾아보다가 SpringBoot의 테스트 실행시에는 `test/resources`에 있는 리소스 파일이 우선적으로 로드된다는 사실을 발견하엿습니다.. 그래서 `test` 하부 `directory`인 `test/rescources`에 테스트전용 `application.properites`를 만들어 테스트를 실행해보니 `application-dev.properties`가 없어도 `test/rescources/application.properites` 가 대신 그 역할을 해주어 테스트가 정상적으로 통과하는것을 확인하였습니다.

이번 이슈로 테스트환경과 운영환경의 단절을 미리 고려하면서 코드를 작성했으면 좋았겠다는 생각이 들었습니다.