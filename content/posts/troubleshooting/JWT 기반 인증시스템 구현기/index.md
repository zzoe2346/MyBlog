---
title: JWT가진 쿠키 인증을위한 JwtFilter 구현기
date: 2025-02-13
tags:
  - JWT
  - Troubleshooting
categories: 트러블슈팅
---

## 구현 배경

- State 인 SESSION 방식과 비교하여 Stateless JWT를 활용하여 서버 부담 완화
- 브라우저에 쿠키가 자동으로 저장되므로 같은 브라우저를 쓴다면 My Coding Test Connector(크롬 익스텐션) 에서도 인증용 쿠키로 사용이 가능

## 구현 내용

- 로그인 성공 시, 서버는 사용자 정보를 담은 JWT 토큰을 생성후, 쿠키에 담아 응답
- 서버는 JwtFilter를 통해 토큰의 유효성을 검증하고, 사용자 정보를 추출하여 인증을 완료
- Spring Security Filter에 구현한 JwtFilter를 추가하여 인증 작업을하도록 구현
- 쿠키에 httpOnly=true, secure=true, sameSite=None 속성을 적용하여 XSS 및 CSRF 공격 방지

다음 그림은 JwtFilter 의 흐름을 간단히 다이어그램으로 표현
![](Pasted%20image%2020250213192721.png)

```java JwtFilter.java 
package com.mycodingtest.security;

import com.mycodingtest.util.JwtUtil;
import io.jsonwebtoken.Claims;
import jakarta.servlet.FilterChain;
import jakarta.servlet.ServletException;
import jakarta.servlet.http.Cookie;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.security.authentication.UsernamePasswordAuthenticationToken;
import org.springframework.security.core.context.SecurityContextHolder;
import org.springframework.security.web.authentication.WebAuthenticationDetailsSource;
import org.springframework.stereotype.Component;
import org.springframework.web.filter.OncePerRequestFilter;

import java.io.IOException;

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