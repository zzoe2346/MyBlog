---
title: "'편한 길'보다 '맞는 길' - SDK 독립성을 지켜낸 다국어 기능 구현 후기"
date: 2025-07-06
tags: 
categories: 
summary: 
draft: "true"
---

## 다국어 기능, 그냥 만들면 되는 거 아니었어?
입사 후 3개월 가까워져 가는 때 다국어 기능 구현을 맡게되었다. 상사에게 기능 요구 사항을 듣고서는 사실 "기존에 하드코딩되어있는 한글 문자열을 코드화만 하면 되겠군"하는 생각이 처음에 들었고 그냥 노가다겠구나하는 생각이 들었다.

GEMINI한테 물어보니 Spring에서는 자체 구현한 ResourceBundle 인터셉터, 리졸버등이 잘되어있었다


## 첫번째 갈림길: Spring의 MessageSource
이걸쓰면 매우 편했다. 간단히 스프링 설정 메서드만 하나 설정해주면 다국어 기능을 지원하는 MessageSource를 빈컨테이너에 넣어주었고 개발자는 그냥 쓰고싶은 클래스에서 의존성 주입을하여 사용하면 되었다.

또한 우리 솔루션은 Spring Boot 기반이기에 의존성 주입이 편했다.

하지만, 내가 생각하기에 여기넹 다음과 같은 이슈가 있었다. 
- SDK에도 적용해야되는데 만약 스프링을 안쓸때는 어떻게하지?
## Common SDK 의 독립성
우리 회사는 현재 2개의 SDK 프로젝트가 존재한다. 하나는 내가 속한 팀만을 위한 SDK, 또다른 하나는 다른 팀과 공통으로 쓸 SDK. 이 공통 SDK에도 다국어를 적용해야할 부분이 있었고 이를 위해 

만약 이 SDK 가 스프링 기능에 의존하면 이 SDK를 쓰는 모든 프로젝트는 다국어기능을 쓰기위해 강제로 스프링에 종속되는 이슈가 있었던것

무지성으로 스프링의 추상화에 의존하지말고 좀 더 고민을 해보기로했다.
## java.servlet.util.i18n

결국 Java에 내장된 ResourceBundle과 MessageFormat을 발견했다. 이것을 사용하면 어떤 자바 프로젝트에서도 동작하는 다국어 기능을 만들 수 있음을 깨달았고 POC를 통해 잘 작동하는것도 신속히 학인을 하였다. 
## 의존성 없는 I18nMessageSource의 탄생
```java
public class I18nMessageSource {
	/**
	 * @see <p>
	 *     지정된 키에 해당하는 포맷팅된 메시지 반환.
	 *     메시지에 {0}, {1} 등의 파라미터가 포함된 경우 사용.
	 *      </p>
	 * @param key 메시지 키
	 * @param args 메시지에 전달할 인자들
	 * @return 포맷팅된 메시지 문자열
	 */
	public static String getMessage(String key, Object... args) {
		return MessageFormat.format(
			ResourceBundle.getBundle("i18n.messages", Locale.getDefaultLocale()).getString(key), args);
	}

	/**
	 * @see <p>지정된 키에 해당하는 메시지 반환.</p>
	 * @param key 메시지 키
	 * @return 메시지 문자열
	 */
	public static String getMessage(String key) {
		return ResourceBundle.getBundle("i18n.messages", Locale.getDefaultLocale()).getString(key);
	}
}
```
- static 메서드를 사용한 이유: 외부에서 객체를 생성하거나 의존성을 주입(DI)할 필요 없이 클래스명.메서드() 형태로 간편하게 호출할 수 있도록 하기 위함.
- 결과적으로 SDK는 이제 스프링 컨테이너와 완전히 독립적인 다국어 기능을 갖게 됨.

## 교훈
시니어 개발자에게 코드 리뷰를 받고 칭찬받았던 일화를 소개.
"단순히 기능을 빠르게 만드는 것보다, 내가 만드는 소프트웨어의 역할과 다른 시스템과의 관계를 고민하는 것이 중요하다는 것을 배웠다."
3개월 차 신입으로서 큰 성장을 이룬 경험이었다는 소감으로 마무리.
