---
title: Oauth2.0 기반 소셜로그인 구현기
date: 2025-02-16
tags:
  - SpringSecurity
  - Oauth
  - Troubleshooting
categories: 트러블슈팅
summary: Spring Security 첫 사용
---

## 문제 정의

- 기존 시스템: Id, Password 방식을 우선 구현. 프론트, 백 모두 구현은 완료된 상태
- Id, Password 방식으로 이메일 인증까지 요구하는 회원가입 절차
- 굳이 이렇게 긴 회원가입을 내 서비스에 요구 할 필요가 있는가?

## 문제 분석

- ⭐️ 회원가입 절차가 너무 김 -> 나라도 듣보 서비스에서는 귀찮아서 그냥 안할거 같음
    - 아이디 중복 체크
    - 비밀번호 안전한지 체크
    - 이메일 인증
- 이메일 시스템을 외부에 의존해야함
- 비밀번호를 DB에 직접 암호화해서 관리

## 문제 해결 과정

- 소셜 로그인으로 전환
- 이전 카카오 테크 캠퍼스에서 공부했던 자료 재공부
  ![](Pasted%20image%2020250216214518.png)
- 저번에는 Intercepter로 구현했었는데 이번에는 Spring Security 를 활용해서 구현 시도
- Spring Security OAuth2 Client를 활용, Google 및 Kakao 소셜 로그인 기능을 구현
- 이번에는 구현에 중심을 두기로함. 한달내에 구현완료하는게 목표이기 때문

proerties에서 구글은 `spring-security-oauth2-client` 에 미리 세팅이 되어있어 편했지만 카카오의 경우는 비교적 설정할게 많았다. 기록용으로 남겨둔다.

```properties
## Google  
spring.security.oauth2.client.registration.google.client-id=?
spring.security.oauth2.client.registration.google.client-secret=?
spring.security.oauth2.client.registration.google.scope=email,profile  
# Kakao  
spring.security.oauth2.client.registration.kakao.client-name=kakao  
spring.security.oauth2.client.registration.kakao.client-id=?  
spring.security.oauth2.client.registration.kakao.client-secret=?  
spring.security.oauth2.client.registration.kakao.client-authentication-method=client_secret_post  
spring.security.oauth2.client.registration.kakao.authorization-grant-type=authorization_code  
spring.security.oauth2.client.registration.kakao.redirect-uri=https://api.mycodingtest.com/login/oauth2/code/kakao  
spring.security.oauth2.client.provider.kakao.authorization-uri=https://kauth.kakao.com/oauth/authorize  
spring.security.oauth2.client.provider.kakao.token-uri=https://kauth.kakao.com/oauth/token  
spring.security.oauth2.client.provider.kakao.user-info-uri=https://kapi.kakao.com/v2/user/me  
spring.security.oauth2.client.provider.kakao.user-name-attribute=id
```

아래 코드는 `AuthenticationSuccessHandler`의 구현체인 `CustomOAuth2SuccessHandler` 이다. security config, properties 를 잘 작성하고 필요한
기능이 있으면 그거에대한 구현체를 만들면 되서 참 편리하게 만들어놓았다는 생각이들었다.

```java CustomOAuth2SuccessHandler.java
@Component  
public class CustomOAuth2SuccessHandler implements AuthenticationSuccessHandler {  
  
    @Value("${url.redirect}")  
    private String redirectUrl;  
    @Value("${cookie.name}")  
    private String cookieName;  
  
    private final UserRepository userRepository;  
    private final JwtUtil jwtUtil;  
  
    public CustomOAuth2SuccessHandler(UserRepository userRepository, JwtUtil jwtUtil) {  
        this.userRepository = userRepository;  
        this.jwtUtil = jwtUtil;  
    }  
  
    @Override  
    public void onAuthenticationSuccess(HttpServletRequest request, HttpServletResponse response, Authentication authentication) throws IOException {  
        OAuth2AuthenticationToken authToken = (OAuth2AuthenticationToken) authentication;  
        OAuth2User oauth2User = authToken.getPrincipal();  
        String provider = authToken.getAuthorizedClientRegistrationId();  
  
        String email;  
        String name;  
        String picture;  
        String oauthId;  
  
        if (provider.equals("google")) {  
            email = oauth2User.getAttribute("email");  
            name = oauth2User.getAttribute("name");  
            picture = oauth2User.getAttribute("picture");  
            oauthId = oauth2User.getAttribute("sub");  
        } else if (provider.equals("kakao")) {  
            Map<String, Object> properties = oauth2User.getAttribute("properties");  
            email = "no mail";  
            name = (String) properties.get("nickname");  
            picture = (String) properties.get("thumbnail_image");  
            oauthId = String.valueOf(oauth2User.getAttributes().get("id"));  
        } else {  
            throw new RuntimeException("지원하는 oauth2 제공자가 아닙니다.");  
        }  
  
        User user = userRepository.findByOauthProviderAndOauthId(provider, oauthId)  
                .orElseGet(() -> userRepository.save(new User(name, email, picture, provider, oauthId)));  
  
        String token = jwtUtil.generateToken(user.getId(), user.getPicture(), name);  
        ResponseCookie cookie = ResponseCookie.from(cookieName, token)  
                .httpOnly(true)  
                .maxAge(60 * 60 * 24 * 90)  
                .path("/")  
                .sameSite("None")  
                .secure(true)  
                .build();  
        response.addHeader(HttpHeaders.SET_COOKIE, cookie.toString());  
        response.sendRedirect(redirectUrl);  
  
        //SecurityContext 즉시 초기화 (세션 제거)  
        SecurityContextHolder.clearContext();  
    }  
}
```

- 유저 식별을 처음에는 이메일을 고려했지만 구글과 다르게 카카오 로그인에서는 이메일을 얻기위해서는 사업자 등록증이 필요하였다. 그래서 결국 유저 식별은 `oauthId`를 활용하였다.

## 개선 결과

- 소셜 로그인 구현 성공
- 유저에게 더 나은 경험 제공 가능
- 회원가입을 생략함으로써 유저 유치 더 원활할 것으로 기대
- `implementation 'org.springframework.security:spring-security-oauth2-client:6.4.2'` 을 사용하니 저번 프로젝트때의 인터셉터때보다 더 생산성있게
  구현이 가능했음

![](SCR-20250216-swak.png#center)
