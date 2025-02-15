---
title: 자료형의 범위와 관련한 return o2 - o1 주의사항
tags:
  - Java
date: 2024-06-04
---
https://www.acmicpc.net/problem/7662
## 문제 정의
주어지는 값이 INTEGER_MAX, INTEGER_MIN 까지 주어진다. 이때
```java
PriorityQueue<Integer> maxQ = new PriorityQueue<>((o1,o2)->{
	return o2-o1;
});
```
이런 코드를 사용해서 내림차순 큐를 구현했는데 의도한대로 코드가 동작하지 않는다.
## 문제 분석
- `o2 = INTEGER_MAX`이고 `o1 = INTEGER_MIN`이면 오버플로가 발생해서 의도하지 않은 결과 도출 원인으로 분석
## 문제 해결 과정
```java
PriorityQueue<Integer> maxQ = new PriorityQueue<>(Comparator.reverseOrder());
```
또는
```java
PriorityQueue<Integer> maxQ = new PriorityQueue<>((o1,o2)->{
	if(o1>o2){
	}else{
	}
});
```

## 개선 결과
- 의도한 대로 오름차순, 내림차순의 우선순위 큐가 동작
## 결론
- 자료형의 최대, 최소 값을 쓸때는 오버플로우, 언더플로우를 고려해야한다.