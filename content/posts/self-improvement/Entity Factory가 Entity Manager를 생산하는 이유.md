---
title: Entity Factory가 Entity Manager를 생산하는 이유
tags:
  - JPA
date: 2024-10-30
categories:
  - 공부
---

## 의문

'자바 ORM 표준 JPA 프로그래밍' 을 읽던중 스프링 애플리케이션 초기에 엔티티 매니저 팩토리가 생성되고 애플리케이션이 종료될 때까지 엔티티 매니저를 생성해 준다고 한다. 왜 엔티티 매니저 팩토리이 필요할까?
그때 그때 매를 만들거나, 엔티티 매니저 풀을 만들어 놓으면 안되는걸까?

## 해소

- 팩은 엔티티 매니저 생명주기에 관여하는 중요한 개념
- 팩이 기억하는 것
    - 데이터베이스 연결 정보
    - 여러 설정들 통합 관리
- 팩토리 패턴
    - 팩토리는 고용 에이전시 역할
    - 직원 고용 하려고 할 때, 우리는 그 사람의 기술, 인성 등등 알게 많다. 그런데 이런걸 알려면 회사차원에서 리소스가 많이든다.
    - 그래서 팩토리(고용 에이전시) 가 이걸 대행해주는 것
    - 팩토리가 있으면 그 팩토리가 뭐에 의존하고 있는지 알 필요 없이 팩토리 사용자가 필요한 객체를 생성해주는것!
- 엔티티 매니저 풀(pool) 이 안쓰이는 이유 고민
    - 미리 설정해 놓을게 많아서 팩토리 패턴을 쓰는것. 그러면 설정 사항 어차피 캐싱해놓고 쓰는거니까
    - 엔티티 매니저는 생성하는데 그렇게 큰 리소스가 들지는 않는거 같다. 왜? 팩토리가 미리 로드해 놓은 정보를 바탕으로 생성이 되니깐.
    - DAO 기능의 협렵에 포함되는 객채가 엔티티 매니저인데 이걸 Pool 로 관리하면 동시성 등 관리하기가 정말 복잡해질 거 같다
    - 풀의 단점은 리소스(메모리 등)을 계속 소유한다는 것. 엔티티 매니저는 한 트랜젝션이 끝나면 필요가 없다.
    - 데이터베이스 커넥션 풀을 관리하는 주체가 엔티티 매니저 팩토리라고 한다.
- 모든 기술은 Trade Off 가 있는 법

## 생각

- 요즘은 스프링 부트로 JPA 를 사용할때 알아서 설정을 해주어서 딱히 엔티티 팩토리 매니저등을 설정안해줘도 된다. 아래코드는 스프링에서 엔티티 팩토리 매니저를 bean화 해주는 설정이다. 문서를 보니 이것말고도
  방식이 더 있더라.

```
<beans>
	<bean id="myEmf" class="org.springframework.orm.jpa.LocalEntityManagerFactoryBean">
		<property name="persistenceUnitName" value="myPersistenceUnit"/>
	</bean>
</beans>
```

- 디자인 패턴중에서 [[Factory]] 패턴 정말 많이 들어보았는데 다음에는 그거에 대해 더 자세히 알아보자.

## 자료

- https://stackoverflow.com/questions/69849/factory-pattern-when-to-use-factory-methods
- https://docs.spring.io/spring-framework/reference/data-access/orm/jpa.html
- https://www.baeldung.com/java-factory-pattern
- https://stackoverflow.com/questions/34580418/why-do-i-need-an-entitymanagerfactory(스프링 부트가 나오고 얼마 안 지나서 등록된 질문. 옛날에는
  xml에 직접 bean을 명시해야해서 뭐 설정할게 많은거 같음)