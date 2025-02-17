---
title: 쿠키 기반 JWT 인증을 위한 Spring Security 필터 구현기
date: 2025-02-13
tags:
  - JWT
  - Troubleshooting
  - SpringSecurity
categories: 트러블슈팅
---
## 문제 정의
- Oauth2.0 로그인 구현 성공
- 로그인시 서버에서 쿠키를 클라이언트에 전달
- **이제 브라우저에서 API를 요청시 전달받은 쿠키를 헤더에 넣어서 요청을 함**
- 서버에서 이제 **쿠키를 확인**하고 **처리**하는 로직을 구현해야한다

## 문제 분석
- 결국 해야하는것은 정상적인 쿠키일때 `Security Context`에 인증이 되었다고 설정해주는 것

## 문제 해결 과정
- 로그인 성공 시, 서버는 사용자 정보를 담은 JWT 토큰을 생성후, 쿠키에 담아 응답
- 서버는 JwtFilter를 통해 토큰의 유효성을 검증하고, 사용자 정보를 추출하여 인증을 완료
- Spring Security Filter에 구현한 JwtFilter를 추가하여 인증 작업을하도록 구현

다음 그림은 JwtFilter 의 흐름을 간단히 다이어그램으로 표현
![](Pasted%20image%2020250213192721.png)

```java JwtFilter.java 
@Component
public class JwtFilter extends OncePerRequestFilter {

    @Value("${cookie.name}")
    private String cookieName;

    private final JwtUtil jwtUtil;

    public JwtFilter(JwtUtil jwtUtil) {
        this.jwtUtil = jwtUtil;
    }

    @Override
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain filterChain) throws ServletException, IOException {
        String token = getJwtFromCookie(request);
        if (token != null) {
            setAuthenticationContext(request, token);
        }
        filterChain.doFilter(request, response);
    }

    private void setAuthenticationContext(HttpServletRequest request, String token) {
        if (SecurityContextHolder.getContext().getAuthentication() == null) {
            Claims claims = jwtUtil.extractAllClaims(token);

            CustomUserDetails customUserDetails = new CustomUserDetails(claims.get("userId", Long.class), claims.get("picture", String.class), claims.get("name", String.class));

            UsernamePasswordAuthenticationToken authToken = new UsernamePasswordAuthenticationToken(customUserDetails, null, null);
            authToken.setDetails(new WebAuthenticationDetailsSource().buildDetails(request));
            SecurityContextHolder.getContext().setAuthentication(authToken);
        }
    }

    private String getJwtFromCookie(HttpServletRequest request) {
        if (request.getCookies() != null) {
            for (Cookie cookie : request.getCookies()) {
                if (cookie.getName().equals(cookieName)) {
                    return cookie.getValue();
                }
            }
        }
        return null;
    }
}
```
## 결과
- 성공적인 인증: 클라이언트에서 API 요청 시, JwtFilter가 쿠키에서 JWT 토큰을 성공적으로 추출하고 검증하여 Security Context에 인증 정보를 설정하는 것을 확인
- 인가되지 않은 리소스 접근 거부: 인증되지 않은 사용자는 접근 시 401 에러를 반환



