---
title: Java Thread 생명 주기
tags:
  - Java
  - Thread
date: 2024-08-26
---
## 생성 상태
생성되었지만 시작되지 않은 스레드. 각 스레드에 `start` 메서드 호출 전까지의 상태
```java
public class NewThread {  
    public void newThread() {  
        // t1 스레드 생성  
        Thread t1 = new Thread(() -> { });  
        System.out.println("New thread t1: " + t1.getState());  
  
        // t2 스레드 생성  
        Runnable runnable1 = () -> { };  
        Thread t2 = new Thread(runnable1);  
        System.out.println("New thread t2: " + t2.getState());  
  
        // t3 스레드 생성  
        Thread t3 = new Thread(new Runnable() {  
            @Override  
            public void run() {  
            }  
        });  
        System.out.println("New thread t3: " + t3.getState());  
  
        // t4 스레드 생성  
        Thread t4 = new Thread(new Thread() {  
            @Override  
            public void run() { }  
        });  
        System.out.println("New thread t4: " + t4.getState() + "\n");  
    }  
}
```
## 실행 대기 상태
`start` 호출 시 스레드가 생성 상태에서 실행 대기(RUNNABLE) 상태로 전환된다. JVM 스레드 스케줄러가 실행에 필요한 리소스와 시간을 할당히기를 기다리는 스레드는 실행할 준비가 되었지만, 아직 실행되지는 않음. CPU 사용 가능하게 되면 스케줄러가 스레드를 실행함.
```java
public class RunnableThread {  
    public void runnableThread() {  
        Thread t1 = new Thread(() -> { });  
        t1.start();  
  
        System.out.println("RunnableThread t1: " + t1.getState());  
  
        Runnable runnable1 = () -> { };  
        Thread t2 = new Thread(runnable1);  
        t2.start();  
        System.out.println("RunnableThread t2: " + t2.getState());  
  
        Thread t3 = new Thread(new Runnable() {  
            @Override  
            public void run() { }  
        });  
        t3.start();  
        System.out.println("RunnableThread t3: " + t3.getState());  
  
        Thread t4 = new Thread(new Thread() {  
            @Override  
            public void run() { }  
        });  
        t4.start();  
        System.out.println("RunnableThread t4: " + t4.getState() + "\n");  
    }  
}
```
## 블록 상태
동기화 블록이나 I/O 작업을 하는 스레드가 BLOCKED 상태에 들어갈 수 있음. 예를 들어 t1이 다른 스레드 t2에서 이미 접근 중인 동기화된 코드 블록에 들어가려하면, t1은 필요한 록을 얻기 전까지 블록 상태로 유지됨
```java
public class BlockedThread {  
    public void blockedThread() {  
        Thread t1 = new Thread(new SyncBlockCode());  
        Thread t2 = new Thread(new SyncBlockCode());  
  
        t1.start();  
  
        try {  
            Thread.sleep(2000);  
        } catch (InterruptedException ex) {  
            Thread.currentThread().interrupt();  
            // 로그 관련 코드 삽입  
        }  
  
        t2.start();  
  
        try {  
            Thread.sleep(2000);  
        } catch (InterruptedException ex) {  
            Thread.currentThread().interrupt();  
            // 로그 관련 코드 삽입  
        }  
  
        System.out.println("Blocked thread t1: "   
+ t1.getState() + "(" + t1.getName() + ")");  
        System.out.println("Blocked thread t2: "   
+ t2.getState() + "(" + t2.getName() + ")");  
  
        System.exit(0);  
    }  
  
    private static class SyncBlockCode implements Runnable {  
        @Override  
        public void run() {  
            System.out.println("Thread "   
+ Thread.currentThread().getName() + " is in run() method");  
            syncMethod();  
        }  
  
        public static synchronized void syncMethod() {  
            System.out.println("Thread "   
+ Thread.currentThread().getName() + " is in syncMethod() method");  
            while (true) {  
                // t1은 여기에 영원히 유지되므로 t2의 접근은 차단됩니다.  
            }  
        }  
    }  
}
```
## 일시 정지 상태(WAITING)
t1에 명시적인 시간 초과 기준을 설정하지 않고 다른 스레드 t2가 작업을 완료하기를 기다릴 때의 상태
```java
/* WAITING(일시 정지) 상태의 예:  
   1. t1 스레드를 생성합니다.  
   2. start 메서드를 통해 t1 실행을 시작합니다.  
   3. t1의 run 메서드에서 다음을 수행합니다.  
      1. 다른 스레드 t2를 생성합니다.  
      2. start 메서드를 통해 t2 실행을 시작합니다.  
      3. t2가 실행되는 동안 t2.join을 호출합니다. t2는 t1에  
         조인(t1은 t2 실행이 종료될 때까지 대기해야 함)해야 하므로  
         t1은 WAITING 상태입니다.  
   4. t2의 run 메서드에서 t2는 t1의 상태를 출력합니다.  
      이 상태는 WAITING이어야 합니다(t1 상태를 출력하는 동안 t2는 실행 중이므로 t1은 대기 중입니다).  
*/  
  
public class WaitingThread {  
    private static final Thread t1 = new CodeT1();  
  
    public void waitingThread() {  
        t1.start();  
    }  
  
    private static class CodeT1 extends Thread {  
        @Override  
        public void run() {  
            Thread t2 = new Thread(new CodeT2());  
            t2.start();  
  
            try {  
                t2.join();  
            } catch (InterruptedException ex) {  
                Thread.currentThread().interrupt();  
                // 로그 관련 코드 삽입  
            }  
        }  
    }  
  
    private static class CodeT2 implements Runnable {  
        @Override  
        public void run() {  
            try {  
                Thread.sleep(2000);  
            } catch (InterruptedException ex) {  
                Thread.currentThread().interrupt();  
                // 로그 관련 코드 삽입  
            }  
  
            System.out.println("WaitingThread t1: " + t1.getState() + " \n");  
        }  
    }  
}
```
## 일시 정지 상태(TIMED_WAITING)
스레드 t1이 다른 스레드 t2가 작업을 완료할 때까지 명시적인 시간을 기다릴때의 상태
```java
/* TIME_WAITING(일시 정지) 상태의 시나리오:  
   1. t1 스레드를 생성합니다.  
   2. start 메서드를 통해 t1 실행을 시작합니다.  
   3. t1의 run 메서드에 2초(임의 시간)의 휴면 시간을 추가합니다.  
   4. t1이 실행되는 동안 메인 스레드는 t1 상태를 출력합니다.  
      t1은 2초 후에 만료되는 sleep 메서드가 관리하므로 상태는 TIME_WAITING이어야 합니다.  
*/  
  
public class TimedWaitingThread {  
    public void timedWaitingThread() {  
        Thread t = new Thread(() -> {  
            try {  
                Thread.sleep(2000);  
            } catch (InterruptedException ex) {  
                Thread.currentThread().interrupt();  
                // 로그 관련 코드 삽입  
            }  
        });  
        t.start();  
  
        try {  
            Thread.sleep(500);  
        } catch (InterruptedException ex) {  
            Thread.currentThread().interrupt();  
            // 로그 관련 코드 삽입  
        }  
  
        System.out.println("TimedWaitingThread t: " + t.getState() + "\n");  
    }  
}
```
## 종료 상태(TERMINATED)
비정상적으로 중단 또는 성공적으로 작업을 완료한 상태
```java
/* TERMINATED(종료) 상태의 시나리오:  
   애플리케이션의 메인 스레드는 스레드의 상태 t를 출력합니다.  
   상태 t를 출력하면 스레드 t가 작업을 완료한 것입니다.  
*/  
  
public class TerminatedThread {  
    public void terminatedThread() {  
        Thread t = new Thread(() -> { });  
        t.start();  
  
        try {  
            Thread.sleep(1000);  
        } catch (InterruptedException ex) {  
            Thread.currentThread().interrupt();  
            // 로그 관련 코드 삽입  
        }  
  
        System.out.println("TerminatedThread t: " + t.getState() + "\n");  
    }  
}
```
