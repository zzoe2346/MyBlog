---
tags:
  - OS
  - Thread
  - JVM
date: 2024-05-19
categories:
  - 공부
title: OS와 JVM에서의 Thread 차이
---

## 자바가 관리하는 스레드 (Java Thread)

- 자바 언어의 쓰레드 모델은 JVM 내에서 관리된다.
- 자바 프로그램에서는 java.lang.Thread 클래스나 java.util.concurrent 패키지의 ExecutorService 및 ForkJoinPool 등을 사용하여 쓰레드를 생성하고 관리.
- 자바 쓰레드는 JVM이 제어하며, 자바 프로그래머가 직접 조작할 수 있다.

## 운영 체제에서 관리하는 스레드 (OS Thread)

- 운영 체제는 하드웨어의 프로세서를 효율적으로 활용하기 위해 쓰레드를 생성하고 관리.
- 운영 체제 쓰레드는 주로 커널 모드에서 실행되며, 운영 체제가 직접 관리.
- 대부분의 운영 체제는 쓰레드를 생성하고 관리하기 위한 시스템 콜을 제공하여 다양한 작업을 수행.

## Reference

- https://www.geeksforgeeks.org/difference-between-java-threads-and-os-threads/