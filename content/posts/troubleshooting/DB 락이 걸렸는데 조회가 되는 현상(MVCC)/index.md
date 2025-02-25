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

## Challenge

> 이 글에서 Lock은 Exclusive Lock을 의미합니다

포인트 차감/증가 로직 개발 중, 포인트가 돈과 같은 중요 자산이므로 데이터 정합성을 확보하기 위해 포인트 데이터에 Lock을 적용하였습니다. Lock 설정 후, 여러 스레드(트랜잭션)에서 동시에 포인트 데이터 접근을 시도하는 상황을 재현하여 Lock 기능의 정상 작동 여부를 검증하는 테스트 코드를 작성하였습니다. 또한 MySQL에서 실행되는 SQL에서도 `FOR UPDATE`가 입력된것을 확인하였습니다. 당시 저는 Lock이되면 다른 스레드(트랜잭션)는 해당 데이터에 접근 시도시 block되거나 예외가 발생하여 rollback될 것이라고 예상했습니다.
```sql
where
    i1_0.id=? for update
```

**하지만** 예상과 다르게 **Lock이 걸린 데이터가 조회**되는 것입니다! 지식의 큰 구멍을 발견하였고 이를 계기로 이 구멍을 채워보기로 결심하였습니다.

- Lock이 걸린 데이터가 조회되는 상황에 대한 원인 파악
- 조회 동작과 Lock 간의 관계 명확히 이해
- 테스트 코드를 활용한 MVCC 동작 검증
## Action

### 원인 파악 & 공부

InnoDB의 MVCC(Multi Versioning Concurrency Control) 기술 때문이라는 것을 파악하였습니다. 공식문서도 찾아보고, LLM한테 물어도보고, 여러 유튜브를 시청하며 공부를 하였습니다. 특히, 쉬운코드님의 유튜브 강의가 학습에 큰 도움이 되었습니다👍 예제를 참 많이 들어주시며 설명하시는데 눈으로만 보지 않고 아래처럼 다이어그램을 그려보니 더 잘 이해가되더군요.

![](Pasted%20image%2020250218193728.png#center)
### Lock-Based Concurrency Control만 가정하면 나의 가정이 맞다
Lock-Based Concurrency Control에서는 최초게 제가 생각처럼 Lock이 되면 조회가 안되고 block이일어나는게 맞습니다.

**하지만** MySQL의 Storage Engine인 InnoDB는 Lock으로 인한 처리량 저하로 인한 성능 저하를 해소하기 위해 MVCC 기법을 동시성 관리를 하는데 추가 하였고, 덕분에 Lock이 걸린 상태의 데이터도 조회가 가능한것입니다.

### MVCC : Lock과 무관한 조회 & 일관성있는 읽기

Lock-Based Concurrency Control에서는 Lock이 걸린 데이터를 조회 시도하면 block이되어 처리량이 줄지만 MVCC는 특정 시점의 데이터베이스 상태를 찍은(스냅샷)것을 활용하여 트랜잭션이 끝날때 까지 그 스냅샷만 조회함으로서 다른 트랜잭션에서 커밋이 되든말든 일관성있게 읽을수 있습니다.(repeatable read이 되는것) 

- MVCC는 커밋된 데이터만 읽는다
- 실시간 최신 데이터보다 특정 시점의 일관된 데이터를 필요로 할 때 MVCC가 유리
- 트랜잭션 실행 중 데이터가 변경되더라도 영향을 받지 않고 안정적인 결과를 얻고 싶을 때

> 스냅샷(snapshot)?
> 
> 특정 시점의 데이터 상태를 기록한 것. 사진을 찍는 것처럼, 트랜잭션이 시작된 순간의 데이터베이스 상태를 "찍어서" 그 트랜잭션이 그 데이터를 기준으로 동작하게 만듬. 트랜잭션이 시작될 때 특정 버전의 데이터를 스냅샷으로 캡처하고, 해당 트랜잭션 내에서는 항상 이 스냅샷을 기준으로 데이터를 읽음
### MVCC 장점/단점
장점
- 읽기 작업(SELECT)이 쓰기 작업(UPDATE, INSERT)을 방해하지 않도록 처리량 높이기 위해 설계된 메커니즘
- 일관성 있는 읽기(Consistent Read) 보장
- dirty read, non repeatable read 문제 해결

단점
- 스냅샷같은 요소때문에 메모리 자원 더 요구리소스 사용 증가: Undo Log와 버전 관리로 저장 공간과 계산 비용 증가
- 실시간성 부족: 스냅샷 기반이라 최신 데이터를 보장하지 않음

### 정합성 보장: 개발자의 역량이 필요
MySQL의 경우, 데이터의 정합성을 보장하고 싶다면 꼭 망각하지 말고 Lock을 걸어주어야 합니다. 

상품 재고, 계좌 이체처럼 **정합성이 매우 중요한 데이터**는 여러 트랜잭션이 동시에 접근하여 수정할 수 있으므로, **Lock을 사용하여 동시성에 의한 이상 현상을 방지**해야 합니다. MySQL의 기본 Isolation Level인 REPEATABLE READ에서는 MVCC가 동작하지만, Lock을 사용하면 MVCC 대신 Lock-Based Concurrency Control이 적용되어 처리량이 감소하는 대신 정합성을 보장할 수 있습니다.

반면, PostgreSQL은 트랜잭션 충돌 시 롤백을 발생시키므로 재시도 로직이 필요하지만, 기본적으로 정합성을 유지하도록 설계되어 있습니다.

### DBMS별 다른 정합성 보장 방법 간단히 알아보기
**MySQL의 경우**
- read 후 update해야할 데이터의 경우
	- 정합성이 중요. isolation level이 repeatable read라도 for update를 안해주면 그냥 진행시켜서 정합성 문제 발생
	- MySQL에서는 read 후 update해야되면 꼭 for update를 해줘야지 정합성 문제가 해결된다. 물론 이러면 block은 생긴다. trade off 를 고려하자 정합성이냐 성능이냐
- read 만할 경우(이때 개발자는 데이터의 최신성이 중요한지 고려)
	- MVCC 를 활용시 최신 커밋된 데이터를 block없이 조회가 가능
	- Lock(ex, for update)를 활용 시 시점상 최신 데이터 조회 보장. 단, block가능성 존재
- MySQL에서는 Isolation Level조절을 통한 정합성 문제해결이 아니라 Lock을 잘 챙겨줘야함

**PostgreSQL의 경우**
-  Isolation Level이 reapeatble read 일때, 같은 데이터에 먼저 update한 tx가 커밋되면 이후 tx는 rollback 된다. rollback으로 이상현상을 방지함. 이후 재시도등의 로직 필요

### 테스트 코드로 MVCC 확인하기
어느정도 MVCC에 관해 공부하고 나서는 테스트 코드를 통해 직접 검증해보았습니다. 쉬운코드님이 제시해준 예제를 직접 재현해보았고, 나만의 테스트 케이스를 설계하여 코드로 테스트를 수행해보았습니다.

`ExecutorService`로 멀티스레드 환경을 보장하였고, sleep으로 장기간 Lock상황을 재현하여 Lock된 데이터를 조회하고 Lock을 건 경우와 안 건 경우의 정합성 관련 이슈도 같이 다루어 보았습니다.


```java
@SpringBootTest  
public class MVCCTest {  
  
    private static int SLEEP_TIME = 100;  
    private static int TOTAL_THREAD_WAIT_TIME = 3;  
  
    @Autowired  
    private ItemRepository itemRepository;  
    @Autowired  
    private ItemService itemService;  
    private Long itemId;  
  
    @BeforeEach  
    public void setup() {  
        Item item = new Item("Test Item", 10);  
        itemId = itemRepository.save(item).getId();  
    }  
  
    //Test Database: MySQL  
    //디폴트 isolation level: repeatable read  
    /**     * t1: 먼저 락을 건다.  
     * t2: t1 이후에 락을 건다.  
     * <p>  
     * 결과:t2는 t1의 락이 회수될때 까지 Blocking 된다! 정합성이 확실히 보장되는 경우임  
     */  
    @Test  
    public void test1() throws InterruptedException {  
        //given  
        ExecutorService executorService = Executors.newFixedThreadPool(2);  
        CountDownLatch latch = new CountDownLatch(2);  
  
        //when  
        executorService.submit(() -> {  
            try {  
                itemService.forUpdateLockAndSubtractOneAfterLockDuring2Sec(itemId);  
            } catch (Exception e) {  
                System.out.println("스레드 1 예외 발생: " + e.getMessage());  
            } finally {  
                latch.countDown();  
            }  
        });  
  
        executorService.submit(() -> {  
            try {  
                Thread.sleep(SLEEP_TIME);  
                itemService.forUpdateLockAndSubtractOne(itemId);  
            } catch (Exception e) {  
                System.out.println("스레드 2 예외 발생: " + e.getMessage());  
            } finally {  
                latch.countDown();  
            }  
        });  
  
        executorService.awaitTermination(TOTAL_THREAD_WAIT_TIME, TimeUnit.SECONDS);  
  
        //then  
        Assertions.assertThat(itemRepository.findById(itemId).get().getQuantity()).isEqualTo(8);  
    }  
  
    /**  
     * t1: 2초동안 락을 소유하고 1 차감  
     * t2: 락이 필요없는 조회 요청  
     * <p>  
     * 결과: 아직 락인 상태에서 t2가 아이템 남은 개수 조회시 10개로 조회되면 MVCC에 의한 일관된 조회 성공  
     * 또한, 결국 남은 아이템 개수는 9개가 되야함.  
     */    @Test  
    public void test2() throws InterruptedException {  
        //given  
        ExecutorService executorService = Executors.newFixedThreadPool(2);  
        CountDownLatch latch = new CountDownLatch(2);  
  
        //when  
        executorService.submit(() -> {  
            try {  
                itemService.forUpdateLockAndSubtractOneAfterLockDuring2Sec(itemId);  
            } catch (Exception e) {  
                System.out.println("스레드 1 예외 발생: " + e.getMessage());  
            } finally {  
                latch.countDown();  
            }  
        });  
        AtomicInteger selectedQuantity = new AtomicInteger(-1);  
        executorService.submit(() -> {  
            try {  
                Thread.sleep(SLEEP_TIME);  
                selectedQuantity.set(itemService.justSelect(itemId));  
            } catch (Exception e) {  
                System.out.println("스레드 2 예외 발생: " + e.getMessage());  
            } finally {  
                latch.countDown();  
            }  
        });  
        executorService.awaitTermination(TOTAL_THREAD_WAIT_TIME, TimeUnit.SECONDS);  
  
        //then  
        Assertions.assertThat(selectedQuantity.get()).isEqualTo(10);  
        Assertions.assertThat(itemRepository.findById(itemId).get().getQuantity()).isEqualTo(9);  
    }  
  
    /**  
     * Dirty Read 가 방지됨!  
     * t1: 락이 없이 아이템 수량 1차감 시도. 차감된 상태로 약 2초동안 실행됨  
     * t2: 락이 없이 조회 요청  
     * 결과: t1이 이미 차감되었어도 본인만의 영역에서 차감하였고, 아직 커밋도 안되었기 때문에  
     * t2는 아이템 수량을 10으로 조회한다  
     */  
    @Test  
    public void test3() throws InterruptedException {  
        ExecutorService executorService = Executors.newFixedThreadPool(2);  
        CountDownLatch latch = new CountDownLatch(2);  
  
        executorService.submit(() -> {  
            try {  
                itemService.justSelectAndAndSubtractOneDuring2Sec(itemId);  
            } catch (Exception e) {  
                System.out.println("스레드 1 예외 발생: " + e.getMessage());  
            } finally {  
                latch.countDown();  
            }  
        });  
  
        AtomicInteger selectedQuantity = new AtomicInteger(-1);  
        executorService.submit(() -> {  
            try {  
                Thread.sleep(100);  
                selectedQuantity.set(itemService.justSelect(itemId));  
            } catch (Exception e) {  
                System.out.println("스레드 2 예외 발생: " + e.getMessage());  
            } finally {  
                latch.countDown();  
            }  
        });  
  
        latch.await(TOTAL_THREAD_WAIT_TIME, TimeUnit.SECONDS);  
  
        //then  
        Assertions.assertThat(selectedQuantity.get()).isEqualTo(10);  
        Assertions.assertThat(itemRepository.findById(itemId).get().getQuantity()).isEqualTo(9);  
    }  
  
    /**  
     * MVCC는 커밋된 데이터만 read한다.  
     * !!! 그런데 이 테스트 흠...  
     * t1: 새로운 아이템 추가하고 2초동안 트랜잭션  
     * t2: t1에의해 아이템이 추가되고나서(아직 커밋안된 상태인것_ 전체 아이템수를 조회한다.  
     * 결과: t2는 t1이 추가전의 아이템 개수를 read한다. MVCC는 커밋된 데이터를 read하기 때문!  
     */    @Test  
    public void test4() throws InterruptedException {  
        //givne  
        ExecutorService executorService = Executors.newFixedThreadPool(2);  
        CountDownLatch latch = new CountDownLatch(2);  
        long originalItemCount = itemRepository.count();  
  
        //when  
        executorService.submit(() -> {  
            try {  
                Item item = new Item("Phantom Item", 5);  
                itemService.saverNewItemDuring2Sec(item);  
            } catch (Exception e) {  
                System.out.println("스레드 1 예외 발생: " + e.getMessage());  
            } finally {  
                latch.countDown();  
            }  
        });  
  
        AtomicLong itemCount = new AtomicLong(-1L);  
        executorService.submit(() -> {  
            try {  
                Thread.sleep(300);  
                itemCount.set(itemRepository.count());  
             } catch (Exception e) {  
                System.out.println("스레드 2 예외 발생: " + e.getMessage());  
            } finally {  
                latch.countDown();  
            }  
        });  
  
        latch.await(3, TimeUnit.SECONDS);  
  
        //then  
        Assertions.assertThat(itemCount.get()).isEqualTo(originalItemCount);  
        Assertions.assertThat(itemRepository.count()).isEqualTo(originalItemCount + 1);  
    }  
  
    /**  
     * 두 트랜잭션을 락없이 동시에 수정시키면 Lost Update 가 발생해서 정합성 문제가 생길 가능성이 존재함  
     * t1: 락없이 수량을 1차감  
     * t2: 락없이 수량을 1차감  
     * 결과: 정상적으로 트랜잭션이 serialize하게 되었다면 8이 되야되는데 lost update가 발생하면 9가 될수있다.  
     */    @Test  
    public void test5() throws InterruptedException {  
        //given  
        ExecutorService executorService = Executors.newFixedThreadPool(2);  
        CountDownLatch latch = new CountDownLatch(2);  
  
        executorService.submit(() -> {  
            try {  
                itemService.justSelectAndAndSubtractOne(itemId);  
            } catch (Exception e) {  
                System.out.println("스레드 1 예외 발생: " + e.getMessage());  
            } finally {  
                latch.countDown();  
            }  
        });  
  
        executorService.submit(() -> {  
            try {  
                itemService.justSelectAndAndSubtractOne(itemId);  
            } catch (Exception e) {  
                System.out.println("스레드 2 예외 발생: " + e.getMessage());  
            } finally {  
                latch.countDown();  
            }  
        });  
  
        latch.await(1, TimeUnit.SECONDS);  
  
        //then  
        Item updatedItem = itemRepository.findById(itemId).get();  
        System.out.println(updatedItem.getQuantity());// 8 또는 9가 나옴. 정합성 충족 불가함. 거의 대부분 결과가 9임  
        Assertions.assertThat(updatedItem.getQuantity()).isLessThanOrEqualTo(9);  
    }  
  
    /**  
     * t1: 롤백되는 긴 트랜잭션  
     * t2: t1에서 일단 값이 하나 빼진후에 조회하면 MVCC로 인  
     * 결과: MVCC는 커밋된 값을 읽기에 롤백되든 말든 어차피 커밋이 안된거라 t2는 10을 읽음. 그리고 최종적으로 t1, t2  
     *      모두 커밋 후에는 아이템 수량은 10으로 변함이 없음.  
     */    @Test  
    public void test6() throws InterruptedException {  
        ExecutorService executorService = Executors.newFixedThreadPool(2);  
        CountDownLatch latch = new CountDownLatch(2);  
  
        executorService.submit(() -> {  
            try {  
                itemService.forUpdateLockAndSubtractOneAfterLockDuring2SecAndRollback(itemId);  
            } catch (Exception e) {  
                System.out.println("스레드 1 롤백 발생: " + e.getMessage());  
            } finally {  
                latch.countDown();  
            }  
        });  
  
        AtomicInteger beforeRollbackCount = new AtomicInteger(-1);  
        executorService.submit(() -> {  
            try {  
                Thread.sleep(100);  
                beforeRollbackCount.set(itemService.justSelect(itemId));  
            } catch (Exception e) {  
                System.out.println("스레드 2 예외 발생: " + e.getMessage());  
            } finally {  
                latch.countDown();  
            }  
        });  
  
        latch.await(6, TimeUnit.SECONDS);  
  
        //then  
        Assertions.assertThat(itemRepository.findById(itemId).get().getQuantity()).isEqualTo(10);  
        Assertions.assertThat(beforeRollbackCount.get()).isEqualTo(10);  
    }  
}
```

## Result
이번 학습을 통해 Lock을 설정하는 것만으로 DBMS가 모든 동시성 문제를 해결해줄 것이라는 안일한 생각을 버리게 되었습니다. 동시에 DBMS를 사용하는 개발자로서 기본적인 지식이 부족했음을 뼈저리게 느꼈습니다. 이러한 이해 없이 동시성 이슈에 노출될 가능성이 큰 상황에서 데이터 정합성이 매우 중요한 기능을 구현했다면, 상상 이상의 버그 발생 가능성을 초래했을 것입니다. 
## 참고자료

- https://dev.mysql.com/doc/refman/8.4/en/innodb-multi-versioning.html
- https://www.youtube.com/watch?v=wiVvVanI3p4&list=PLcXyemr8ZeoREWGhhZi5FZs6cvymjIBVe&index=19&ab_channel=%EC%89%AC%EC%9A%B4%EC%BD%94%EB%93%9C