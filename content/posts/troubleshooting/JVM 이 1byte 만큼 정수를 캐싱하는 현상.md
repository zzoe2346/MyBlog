---
title: JVM 이 1byte 만큼 정수를 캐싱하는 현상
tags:
  - JVM
  - Cache
date: 2024-11-12
category: 배움
---

## 어라? 왜 `==` 를 사용했는데 `Long`끼리 제대로 비교가 되지?
Long long1 = 1L;
Long long2 = 2L;
long2 == long2 가 true 가 나오는것을 발견하였다. 만약 이것이 `Stirng` 이라면 납득이 간다. 왜냐하면 `String` 은 `String Pool`이란것을 힙영역에 JVM이 만들어 주지 않는가? 하지만 Long 은 그런게 없는것으로 안다. 왜 이런 현상이 벌어지는 걸까?
## JVM 의 -127 ~ 128 캐싱
JVM이 -127 ~ 128 까지의 정수(Integer, Long)은 힙영역에 캐싱하여 같은 수면 같은 참조를 할당해주는다는 사실을 검색을 통해 알게되었다. 

왜 캐싱을 해주는 것일까? 바로 아래에 답이 있다.

This is based on the assumption that these small values occur much more often than other ints and therefore it makes sense to avoid the overhead of having different objects for every instance (an `Integer` object takes up something like 12 bytes).

-127 ~ 128 정수는 자주 사용이 되므로 공통 인스턴스를 활용해서 새로운 인스턴스 생성을 방지하는것이 더욱 이득이라는 생각에 기반한것이다.

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