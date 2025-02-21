---
title: DB Lock이 걸렸는데 조회가 되는 현상(MVCC)
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

포인트 차감/증가 로직 개발 중, 포인트가 돈과 같은 중요 자산이므로 데이터 정합성을 확보하기 위해 포인트 데이터에 Lock을 적용하였습니다. Lock 설정 후, 여러 스레드(트랜잭션)에서 동시에 포인트 데이터 접근을 시도하는 상황을 재현하여 Lock 기능의 정상 작동 여부를 검증하는 테스트 코드를 작성하였습니다. Lock 설정 로그("FOR UPDATE")도 정상적으로 확인완료하였습니다. 당시 저는 Lock이 설정되면 다른 스레드는 해당 데이터에 접근 자체가 불가능하며, 접근 시도시 block되거나 예외가 발생하여 rollback될 것이라고 예상했습니다.
```sql
where
    i1_0.id=? for update
```

**하지만** 예상과 다르게 **Lock이 걸린 데이터가 조회**되는 것입니다! 지식의 큰 구멍을 발견하였고 이를 계기로 이 구멍을 채워보기로 결심하였습니다.

## Target

- Lock이 걸린 데이터가 조회되는 상황에 대한 원인 파악
- 조회 동작과 Lock 간의 관계 명확히 이해
- 테스트 코드를 활용한 MVCC 동작 검증
## Action

> MVCC: 읽기 작업은 Lock을 걸지 않고, 데이터의 스냅샷을 사용하여 일관성을 유지하는 기술
### 원인 파악 & 공부
처음에는 Chat GPT에 질문하거나 인터넷 검색을 통해 몇 시간 동안 원인을 찾으려 하였다. 몇몇 블로그에서는 나와 동일한 의문을 제기했고, Chat GPT 역시 조회가 불가능해야 한다는 답변이 나오기도하였다.(내 질문이 별로인지 대답이 오Lock가Lock해 더 혼란스러웠다) 테스트 코드에 문제가 있나 싶어 새로운 코드를 작성하고 설정을 변경하는 등 다양한 시도를 해봤지만, Lock이 걸린 데이터가 조회되는 현상은 계속 재현되었다.

결국 우여곡절 끝에 원인이 InnoDB의 MVCC(Multi Versioning Concurrency Control) 기술 때문이라는 것을 파악하였다. 나름 공식문서도 찾아보고, LLM한테 물어도보고, 여러 유튜브를 활용하였는데 특히, 쉬운코드님의 유튜브 강의가 학습에 큰 도움이 되었다. 영상에서 들어주시는 예제가 듣기만해서는잘 이해가안가 밑의 시퀀스 다이어그램으로 이해를 도왔다.

![](Pasted%20image%2020250218193728.png#center)
### Lock-Based Concurrency Control만 가정하면 나의 가정이 맞다
Lock-Based Concurrency Control에서는 먼저 내가 떠올린것처럼 Lock이 되면 조회가 안되고 block이일어나는게 맞다. 

**하지만** 현대의 DBMS들은 Lock으로 인한 처리량 저하로 인한 성능 저하를 해소하기 위해 MVCC(Multiversion Concurrency Control)라는 기법을 동시성 처리하는데 추가 하였고, 이 기법덕에 Lock이 걸린 상태의 데이터도 조회가 가능한것이다.  

### MVCC : Lock과 무관한 조회 & 일관성있는 읽기

Lock-Based Concurrency Control에서는 exclusive lcok이 걸린 데이터를 조회 시도시 block이되어 처리량이 줄지만 mvcc는 특정 시점의 데이터베이스 상태를 찍은(스냅샷)것을 활용하여 트랜잭션이 끝날때 까지 그 스냅샷만 조회함으로서 다른 트랜잭션에서 커밋이 되든말든 일관성있게 읽고(repeatable read이 되는것) Lock이 걸린것도 신경을 안써도 되는것이다.

- MVCC는 커밋된 데이터만 읽는다
- 실시간 최신 데이터보다 특정 시점의 일관된 데이터를 필요로 할 때 MVCC가 유리
- 트랜잭션 실행 중 데이터가 변경되더라도 영향을 받지 않고 안정적인 결과를 얻고 싶을 때

> 스냅샷(snapshot)?
> 
> 특정 시점의 데이터 상태를 기록한 것. 사진을 찍는 것처럼, 트랜잭션이 시작된 순간의 데이터베이스 상태를 "찍어서" 그 트랜잭션이 그 데이터를 기준으로 동작하게 만듬. 트랜잭션이 시작될 때 특정 버전의 데이터를 스냅샷으로 캡처하고, 해당 트랜잭션 내에서는 항상 이 스냅샷을 기준으로 데이터를 읽음
### MVCC 장점/단점
**장점**
- 읽기 작업(SELECT)이 쓰기 작업(UPDATE, INSERT)을 방해하지 않도록 처리량 높이기 위해 설계된 메커니즘
- 일관성 있는 읽기(Consistent Read) 보장
- dirty read, non repeatable read 문제 해결

**단점**
- 스냅샷같은 요소때문에 메모리 자원 더 요구리소스 사용 증가: Undo Log와 버전 관리로 저장 공간과 계산 비용 증가
- 실시간성 부족: 스냅샷 기반이라 최신 데이터를 보장하지 않음

### 정합성 보장: 개발자의 역량이 필요
MySQL의 경우, 데이터의 정합성을 보장하고 싶다면 꼭 망각하지 말고 exclusive lock을 걸어주어야 합니다. (참고로 MySQL, PostgreSQL의 경우 정합성을 보장하는 방식이 약간 다릅니다.) 

상품 재고, 계좌 이체 등 같이 정합성이 매우 중요하면서 여러 트랜잭션이 동시에 접근하여 데이터 수정이 가능하다면 꼭 조치가 더 필요합니다. isolation level이 default인 repeatable read에서 MySQL의 경우는 exclusive lock을 해주면 MVCC가 무시하고 Lock-Based Concurrency Control이되어 처리량에 손해로인한 성능을 다소 희생하여 정합성을 지키고, PostgreSQL의 경우는 롤백을 시킴으로서 재시도로직이 필요하지만 정합성은 지키도록 설계되어 있습니다

### 더 자세히 DBMS별 다른 정합성 보장 방법 
**MySQL의 경우**
- read 후 update해야할 데이터의 경우
	- 정합성이 중요. isolation level이 repeatable read라도 for update를 안해주면 그냥 진행시켜서 정합성 문제가 생긴다.
	- MySQL에서는 read 후 update해야되면 꼭 for update를 해줘야지 정합성 문제가 해결된다. 물론 이러면 block은 생긴다. trade off 를 고려하자 정합성이냐 성능이냐
- read 만 이때 개발자는 데이터의 최신성이 중요한시 고려해야할 듯 하다.
	- MVCC 를 활용하려면 그냥 조회를 하면 최신 커밋된 데이터를 block없이 조회가 가능
	- for update를 활용하면 시점상 가장 최신의 데이터 조회 보장이 된다. 단, block가능성 존재
- MySQL에서는 isolation level조절을 통한 정합성 문제해결이 아니라 for update를 잘 챙겨줘야함

**PostgreSQL의 경우**
-  reapeatble read 일때, 같은 데이터에 먼저 update한 tx가 커밋되면 이후 tx는 rollback 된다. rollback으로 이상현상을 방지함. 재시도가 발생하긴함

### 테스트 코드로 MVCC 확인하기
그리고 어느정도 MVCC에 관해 공부하고 나서는 테스트 코드를 통해 직접 검증하는 과정을 거쳐보았다. 쉬운코드님이 제시해준 예제를 직접 재현해보았고, 나만의 테스트 케이스를 설계하여 직접 코드로 테스트를 수행해
보았다.

`ExecutorService`로 멀티스레드 환경을 보장하였고, sleep으로 장기간 Lock상황을 재현하여 Lock된 데이터를 조회하고 pesiimistic lock을 쓴경우와 안쓴경우의 정합성 관련 이슈도 같이 다루어 보았다.

테스트 코드 목적: 각 테스트 코드가 어떤 상황을 검증하기 위한 것인지 명확하게 설명해주세요. 예를 들어, "t1, t2 둘다 Lock을 필요로 하는 메서드를 실행" 테스트는 FOR UPDATE Lock이 제대로 동작하는지 검증하기 위한 것이라는 점을 명시하는 것이 좋습니다.

테스트 코드 결과: 각 테스트 코드의 결과를 간략하게 요약해주세요. 예상대로 동작했는지, 아니면 예상과 다른 결과가 나왔는지 설명하면 독자들이 테스트의 의미를 더욱 쉽게 이해할 수 있습니다.

예를 들자면 아래와 테스트 코드로 실시하였다. 이 [링크]()에 방문하면 관련된 모든 코드를 볼 수 있다.

```java
@Test  
@DisplayName("t1, t2 둘다 Lock을 필요로 하는 메서드를 실행. t1이 먼저 Lock을 가졌을때 t2는 t1이 Lock을 반환할 때까지 Blocking 된다. 여기서 동시성 관련 문제는 발생하지 않는다. 어떻게 보면 Serializing 방식과 유사.")  
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
            Thread.sleep(100); // 첫 번째 트랜잭션이 Lock 가지는데 성공하도록 대기  
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
이번 학습을 통해 Lock을 설정하면 DBMS가 모든 것을 알아서 처리해줄 것이라는 안일한 생각을 버리게 되었습니다. 동시에 DBMS를 사용하는 개발자로서 기본적인 지식이 부족했음을 깨닫게 되었습니다. 만약 이러한 지식 없이 재고 관리 시스템을 개발했다면, 동시에 여러 사용자가 동일한 상품을 구매하여 재고 부족 사태가 발생하거나, 금액 계산 오류로 인해 금전적 손실이 발생할 수 있었을 것입니다.

테스트 코드를 통해 다양한 상황을 검증하는 방식이 매우 효과적이었다. 당장 내가 작성한 비즈니스 로직에 대한 테스트 코드가 아니라 이렇게 상황을 직접 만들고 테스트 케이스를 만들어 테스트를 하니 고민하던것이 깔끔히 해결되는 느낌이다. 그냥 책이나 인터넷으로 이론적으로 아는건 사실 좀 불안한게 사실인데 이렇게 직접 코드를실행하여 MVCC, 동시성 관련한 문제가 내 예상되로 진핸되는 코드를 작성하니 뿌듯하고 깔끔하여 좋다.
## 참고자료

- https://dev.mysql.com/doc/refman/8.4/en/innodb-multi-versioning.html
- https://www.youtube.com/watch?v=wiVvVanI3p4&list=PLcXyemr8ZeoREWGhhZi5FZs6cvymjIBVe&index=19&ab_channel=%EC%89%AC%EC%9A%B4%EC%BD%94%EB%93%9C