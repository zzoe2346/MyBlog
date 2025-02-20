---
title: DB 락이 걸렸는데 조회가 되는 현상(MVCC)
date: 2025-02-18
tags:
  - MVCC
  - InnoDB
  - DB
  - MySQL
categories: 공부
summary: MVCC 공부
---

## Situation

비관적 락을 걸고 이를 검증하기 위해 테스트 코드를 작성하던 중 이상한 현상을 발견하였다. 분명 비관적 락을 설정했고, 로그에도 "FOR UPDATE"라는 락 설정 표시가 보였다.

```sql
where
        i1_0.id=? for update
```

하지만 예상과 달리, 락이 걸린 데이터가 조회되는 것이 아닌가? 당시 나는 락이 걸리면 어떤 스레드도 해당 데이터에 접근할 수 없다고 생각했고, 조회 시도시 Block되거나 예외가 발생하여 Rollback될 것으로
예상하였다. 그러나 실제로는 아무런 문제 없이 조회가 진행되어 당황스러운 상황이었다.

## Target

- 비관적 락이 걸린 데이터가 조회되는 상황에 대한 원인 파악

## Action

> MVCC: 읽기 작업은 락을 걸지 않고, 데이터의 스냅샷을 사용하여 일관성을 유지하는 기술

처음에는 Chat GPT에 질문하거나 인터넷 검색을 통해 몇 시간 동안 원인을 찾으려 하였다. 몇몇 블로그에서는 나와 동일한 의문을 제기했고, Chat GPT 역시 조회가 불가능해야 한다는 답변이 나오기도하였다.(내 질문이 별로인지 대답이 오락가락해 더 혼란스러웠다) 테스트 코드에 문제가 있나 싶어 새로운 코드를 작성하고 설정을 변경하는 등 다양한 시도를 해봤지만, 락이 걸린 데이터가 조회되는 현상은 계속 재현되었다.

결국 우여곡절 끝에 원인이 InnoDB의 MVCC(Multi Versioning Concurrency Control) 기술 때문이라는 것을 파악하였다. InnoDB는 MySQL에서 지원하는 엔진 중 하나로, 기본 설정되어 있으며 가장 많이 사용되는 엔진이다. 나름 공식문서도 찾아보고 여러 유튜브를 활용하였는데 특히, 쉬운코드님의 유튜브 강의가 학습에 큰 도움이 되었다. 이해가 잘 안되어서 예제를 통해 설명해주시는 내용을 이렇게 시퀀스 다이어그램으로 직접 만들어 보니 이해가 잘 되었다.
![](Pasted%20image%2020250218193728.png#center)

그리고 어느정도 MVCC에 관해 공부하고 나서는 테스트 코드를 통해 직접 검증하는 과정을 거쳐보았다. 쉬운코드님이 제시해준 예제를 직접 재현해보았고, 나만의 테스트 케이스를 설계하여 직접 코드로 테스트를 수행해
보았다.

아쉬운점이 있다면 lost update 같은 

대표적인 전략은 이렇다.

1. Thread 1은 특정 데이터를 비관적 락을 통해 락을 걸고 sleep을 활용하여 계속 락을 유지한다.
2. 이제 Thread 2가 Thread 1이 락을 건 데이터에 여러가지 접근을 해보고 assertJ 같은 도구로 검증을 시도한다. 차감하는 로직을 중간에 끼워 넣어 동시성 테스트도 같이 가능하다.

예를 들자면 아래와 테스트 코드로 실시하였다. 이 [링크]()에 방문하면 관련된 모든 코드를 볼 수 있다.

```java
@Test  
@DisplayName("t1, t2 둘다 락을 필요로 하는 메서드를 실행. t1이 먼저 락을 가졌을때 t2는 t1이 락을 반환할 때까지 Blocking 된다. 여기서 동시성 관련 문제는 발생하지 않는다. 어떻게 보면 Serializing 방식과 유사.")  
public void testTx1LockTx2Read() throws InterruptedException {  
    //given  
    ExecutorService executorService = Executors.newFixedThreadPool(2);  
    CountDownLatch latch = new CountDownLatch(2);  
  
    executorService.submit(() -> {  
        try {  
            itemService.forUpdateLockAndSubtractOneAfterLockDuring5Sec(itemId,1);  
        } catch (Exception e) {  
            System.out.println("스레드 1 예외 발생: " + e.getMessage());  
        } finally {  
            latch.countDown();  
        }  
    });  
  
    executorService.submit(() -> {  
        try {  
            Thread.sleep(100); // 첫 번째 트랜잭션이 락 가지는데 성공하도록 대기  
            itemService.forUpdateLockAndSubtractOne(itemId,2);  
        } catch (Exception e) {  
            System.out.println("스레드 2 예외 발생: " + e.getMessage());  
        } finally {  
            latch.countDown();  
        }  
    });  
  
    //when  
    executorService.awaitTermination(6, TimeUnit.SECONDS);  
  
    //then  
Assertions.assertThat(itemRepository.findById(itemId).get().getQuantity()).isEqualTo(8);  
}
```

## Result

- 이번 학습을 통해 락을 설정하면 DBMS가 모든 것을 알아서 처리해줄 것이라는 안일한 생각을 버리게 되었다. 동시에 DBMS를 사용하는 개발자로서 기본적인 지식이 부족했음을 깨닫게 되었다. 만약 이러한 지식 없이
  비즈니스에 중요한 데이터를 관리했다면 정말 최악의 상황을 맞이했을것이다.
- 앞으로도 공식 문서를 기반으로 하고 "MySQL 2.0"과 같은 서적을 통해 꾸준히 공부하여 최소한 MySQL이 작동하는 로직을 전체적으로 가볍게라도 훑어볼 필요성을 느끼게 되었다.
- 테스트 코드를 통해 다양한 상황을 검증하는 방식이 매우 효과적이었다. 당장 내가 작성한 비즈니스 로직에 대한 테스트 코드가 아니라 이렇게 상황을 직접 만들고 테스트 케이스를 만들어 테스트를 하니 고민하던것이
  깔끔히 해결되는 느낌이다. 그냥 책이나 인터넷으로 이론적으로 아는건 사실 좀 불안한게 사실인데 이렇게 직접 코드를실행하여 MVCC, 동시성 관련한 문제가 내 예상되로 진핸되는 코드를 작성하니 뿌듯하고 깔끔하여
  좋다.

## 참고자료

- https://dev.mysql.com/doc/refman/8.4/en/innodb-multi-versioning.html
- https://www.youtube.com/watch?v=wiVvVanI3p4&list=PLcXyemr8ZeoREWGhhZi5FZs6cvymjIBVe&index=19&ab_channel=%EC%89%AC%EC%9A%B4%EC%BD%94%EB%93%9C