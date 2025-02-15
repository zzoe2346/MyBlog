---
title: "@Configuartion의 원리"
tags:
  - Spring
date: 2024-08-28
---
## 공부 계기
이전 플젝 보는데 `@` 이거 붙이면 어떤원리로 설정이 되는지 의문

## 자료 조사

`@Configuration` 애너테이션이 붙은 클래스는 Spring의 **설정 클래스**로, 애플리케이션 컨텍스트가 초기화될 때 빈(Bean)을 정의하고 설정하는 데 사용됩니다. `@Configuration` 클래스는 다음과 같은 방식으로 빈을 생성하고 관리합니다:

### 1. **빈 정의 및 생성**

- **빈 등록**: `@Configuration` 애너테이션이 붙은 클래스는 Spring 컨테이너에 의해 빈 정의를 포함하는 설정 클래스로 인식됩니다. 이 클래스 내에 `@Bean` 애너테이션이 붙은 메서드를 사용하여 빈을 정의합니다.
- **빈 메서드**: `@Bean` 애너테이션이 붙은 메서드는 해당 메서드가 반환하는 객체를 Spring 컨테이너의 빈으로 등록합니다. 이 메서드는 애플리케이션 컨텍스트가 초기화될 때 호출됩니다.

### 2. **빈 생성 시점**

- **컨테이너 초기화**: Spring 애플리케이션 컨텍스트가 초기화될 때, `@Configuration` 클래스가 먼저 로드되어 빈 정의가 생성됩니다. 이는 애플리케이션 컨텍스트가 시작되기 전에 설정된 모든 빈이 준비되도록 보장합니다.
- **순서**: 일반적으로 `@Configuration` 클래스는 애플리케이션 컨텍스트가 생성될 때 가장 먼저 로드되어 빈을 정의합니다. 이를 통해 애플리케이션의 다른 컴포넌트가 이 빈을 주입받을 수 있도록 합니다.

### 3. **빈 초기화 및 주입**

- **빈 초기화**: `@Configuration` 클래스에서 정의된 빈은 컨테이너 초기화 과정 중에 생성되며, 이후 다른 빈에서 의존성 주입을 통해 사용됩니다.
- **의존성 주입**: 애플리케이션 컨텍스트가 초기화된 후, 정의된 빈들이 서로 주입되거나 사용할 준비가 됩니다. 이를 통해 다양한 서비스와 컴포넌트가 상호작용할 수 있습니다.

### 예시로 설명

#### `RestClientBuilderConfig` 클래스

java

코드 복사

`@Configuration public class RestClientBuilderConfig {      private static final int CONNECT_TIMEOUT_MILLISECOND = 2000;     private static final int READ_TIMEOUT_MILLISECOND = 2000;      @Bean     public RestClient.Builder restClientBuilder() {         return RestClient.builder()                 .requestFactory(getClientHttpRequestFactory());     }      private ClientHttpRequestFactory getClientHttpRequestFactory() {         SimpleClientHttpRequestFactory factory = new SimpleClientHttpRequestFactory();         factory.setConnectTimeout(Duration.ofMillis(CONNECT_TIMEOUT_MILLISECOND));         factory.setReadTimeout(Duration.ofMillis(READ_TIMEOUT_MILLISECOND));          return factory;     } }`

- 이 클래스는 Spring 컨테이너에 의해 초기화될 때 `RestClient.Builder` 빈을 생성하고 설정합니다.
- `restClientBuilder()` 메서드가 호출되어 `RestClient.Builder` 객체가 생성되고, 이는 `RestClient` 설정에 필요한 `ClientHttpRequestFactory`를 포함합니다.
- 이 빈은 애플리케이션의 다른 컴포넌트에서 의존성 주입을 통해 사용할 수 있습니다.

### 결론

`@Configuration` 애너테이션을 사용하면 Spring 애플리케이션 컨텍스트가 초기화될 때 설정된 빈들이 먼저 생성되고 준비되며, 이를 통해 애플리케이션의 다른 부분에서 이 빈들을 사용할 수 있도록 합니다. 이 과정에서 빈의 정의와 생성이 보장되므로, DI(Dependency Injection)와 같은 기능이 올바르게 작동할 수 있습니다.