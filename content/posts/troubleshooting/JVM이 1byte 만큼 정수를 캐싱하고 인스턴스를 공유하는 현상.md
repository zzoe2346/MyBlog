---
title: JVM이 1byte 만큼 정수를 캐싱하고 인스턴스를 공유하는 현상
tags:
  - JVM
  - Cache
  - Troubleshooting
date: 2024-11-12
categories: 트러블슈팅
summary: 잘 작동되던 API가 memberId를 바꾸었더니 에러가 발생한다. 그 원인분석과 테스트를 해본다.
---
## 문제 상황

팀원이 API 요청을 하였고, 브라우저에서 요청값을 확인하면 제대로 서버로 값을 보내는것은 잘 확인이 되는상황에서 400응답이 발생하는것을 카톡에 공유하였습니다.

이전까지는 잘 작동하던 API였고 관련작업도 이후에 딱히 한적이 없는데 이런현상이 발생하여 당황스러웠습니다. 

또한 이상한게 이 API에는 가입(인증)된 member 계정으로 요청해야하는데 초창기 테스트용 member로 요청하면 정상작동이 되었고 최근에 생성한 테스트용 member로 요청시 에러가 발생하는것이었습니다.
## 원인 파악
저희 프로젝트는 클라이언트 - 스프링 애플리케이션 인스턴스 - MySQL 인스턴스 구성이었습니다. 우선 클라이언트단과 DB단을 먼저 분석해보았습니다.
1. 클라이언트 단
	- 브라우저의 개발자 모드에서 서버로 전송되는 header, body값 확인
		- 정상적으로 값을 넣어 보내는것 확인
	- 항상 테스트 해보던 계정으로 문제의 API 요청
		- 정상적인 응답 확인
2. Database 단
	- 클라이언트 단에서 오는 id 값이 실존하는지 확인
		- 실제로 저장이 된것을 확인

이렇게 아무이상이 없는것을 확인하였고 결국 스프링 애플리케이션의 문제임을 파악하게 되었습니다. 조건문에 논리적 오류가 있는지 컨트롤러에서 제대로 요청을 가지고 파라미터로 넘기는지 등등 팀원들과 고민도중 한 팀원분이 `Long` 비교시 `==`가 아닌 `equals()` 로 수정하자 정상작동을 하는것을 확인해주셨습니다.
> callback.getAssignedMemberId(), memberId 는 Long

**기존 코드**
```java
if (callback.getAssignedMemberId() != memberId && callback.getStatus() != Callback.Status.WAITING.name()) {
    throw new BadRequestException("해당 콜백은 대기 상태가 아니고, 배정 시니또가 아닙니다.");
} 
```

**수정 코드**
```java 수정 코드
if (!callback.getAssignedMemberId().equals (memberId) && callback.getStatus() != Callback.Status.WAITING.name()) {
    throw new BadRequestException ("해당 콜백은 대기 상태가 아니고, 배정 시니또가 아닙니다.");
}
```



정말 이상했습니다. 왜 어떤 memberId는 되고 다른건 안되었던걸까요?  equals를 하면 왜 정상적으로 작동을 했던걸까요? 그런데 코드를 보니 애초에 **왜 `==` 를 사용했는데 `Long`끼리 제대로 비교가 정상적으로 작동되고는 했을까요?**

당시에 만약 이것이 `Stirng` 이라면 납득이 갈 수 있습니다. 왜냐하면 `String` 은 `String Pool`이란것을 힙영역에 JVM이 만들어 주는걸로 알고있었기 때문입니다. 하지만 이건 `Long` 입니다.

**원인은 바로 JVM의 캐싱 전략 때문이었습니다!** JVM은 1byte크기의  -127 ~ 128값을 미리 힙역역에 인스턴스화 해놓고  -127 ~ 128해당하는 값들은 이 인스턴스의 레퍼런스를 공유한다고 합니다.

Long은 객체라서 == 같은 비교를 쓰면 레퍼런스값을 비교하게 되는데 -127 ~ 128 은 모든 Long 객체가 같은 레퍼런스를 공유했기에 이런현상이 나왔던것이었습니다! 초창기의 member는 memberId가 128 이하였어서 제대로 비교가 되었고, 최근에 생성된 member의 memberId가 128을 초과하여 레퍼런스 비교에서 무조건 `false`가 발생되여 `throw new BadRequestException` 가 된것이었습니다.
## API가 실패한 이유 정리
- 초창기의 member는 memberId가 128 이하였어서 제대로 비교가 되어 if문이 의도대로 작동
- 하지만, 최근에 생성된 member의 memberId가 128을 초과하여 레퍼런스 비교에서 무조건 `false`가 발생
	- 무조건 `throw new BadRequestException` 되어 에러응답 발생

**결국** API가 정상 작동했다 안했다가한게 초기에 만듣 멤버들을 1L, 2L이렇게 미리 공유가 되는 참조라서 정상적으로 비교가 되었던 것이고 이후 memberId가 128을 초과하게되어 같은 참조를 공유안하고 각 Long 객체마다 다른 레퍼런스값을 가지게되어 이런 현상이 발생한것을 파악하게되었습니다.

## 왜 이렇게 작동하게 만들어 놓았는가?

```
This is based on the assumption that these small values occur much more often than other ints and therefore it makes
sense to avoid the overhead of having different objects for every instance (an`Integer`object takes up something like 12
bytes).
```

-127 ~ 128 정수는 자주 사용이 되므로 공통 인스턴스를 활용해서 새로운 인스턴스 생성을 방지하는것이 더욱 이득이라는 생각에 기반한것이었습니다.

저는 진짜인지 직접 확인하고자 했고 테스트 코드를 작성하여 확인해보았습니다. 아래는 작성된 테스트 코드이고 실제로 인스턴스를 -127 ~ 128 까지는 공유하는것을 확인하였습니다. 이는 Long뿐만 아니라 Integer도 똑같았습니다. Byte, Short 등도 마찬가지라 합니다.
## 테스트 코드를 통한 검증

```java
public class IntegerCachingTest {  
    /**  
     *  Constant Pool: JVM은 -128에서 127 범위의 정수(Short, Integer, Long 등)를 미리 캐싱하여, 같은 값을 반복해서 생성하지 않고, 해당 범위의 정수는 동일 객체를 재사용  
     *   동일한 숫자 객체를 자주 사용하므로 메모리 사용량을 줄이고, 객체 생성 오버헤드를 줄이는 효과  
     */  
    @Test  
    @DisplayName("JVM 의 Integer 캐싱 테스트")  
    void integerCacheTest() {  
        Integer integer1 = 127;  
        Integer integer2 = 127;  
  
        assertTrue(integer1 == integer2);  
  
        integer1 = -128;  
        integer2 = -128;  
  
        assertTrue(integer1 == integer2);  
  
         integer1 = 128;  
         integer2 = 128;  
  
        assertFalse(integer1 == integer2);  
  
        integer1 = -129;  
        integer2 = -129;  
  
        assertFalse(integer1 == integer2);  
    }  
  
    @Test  
    @DisplayName("JVM 의 Long 캐싱 테스트")  
    void longCacheTest() {  
        Long long1 = 127L;  
        Long long2 = 127L;  
  
        assertTrue(long1 == long2);  
  
        long1 = -128L;  
        long2 = -128L;  
  
        assertTrue(long1 == long2);  
  
        long1 = 128L;  
        long2 = 128L;  
  
        assertFalse(long1 == long2);  
  
        long1 = -129L;  
        long2 = -129L;  
  
        assertFalse(long1 == long2);  
    }  
}
```

## 참고자료

- https://www.geeksforgeeks.org/java-integer-cache/
- https://stackoverflow.com/questions/834961/is-the-cpu-wasted-waiting-for-keyboard-input-generic