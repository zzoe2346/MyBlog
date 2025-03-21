---
title: 행 우선 순회 VS 열 우선 순회
tags:
  - Java
  - Memory
date: 2024-06-13
categories:
  - 공부
---

자바에서 2차원 배열을 순회할 때, 배열의 캐싱 동작은 JVM과 하드웨어의 캐시 메모리 구조에 따라 달라집니다. 자바는 배열을 행 우선(row-major) 방식으로 저장합니다. 따라서 행 우선 순회(row-major
order)를 사용하는 것이 메모리 접근 패턴에 더 유리합니다.

### 행 우선 순회 vs. 열 우선 순회

#### 행 우선 순회 (Row-major order)

```java
for (int i = 0; i < rows; i++) {
    for (int j = 0; j < cols; j++) {
        process(array[i][j]);
    }
}
```

#### 열 우선 순회 (Column-major order)

```java
for (int j = 0; j < cols; j++) {
    for (int i = 0; i < rows; i++) {
        process(array[i][j]);
    }
}
```

### 행 우선 순회가 유리한 이유

1. **메모리 연속성**: 2차원 배열이 내부적으로 1차원 배열로 저장될 때, 행 우선 방식으로 메모리에 배치됩니다. 따라서 행 우선 순회는 연속적인 메모리 접근을 의미하며, 이는 CPU 캐시 히트(cache
   hit)를 증가시키고, 메모리 접근 속도를 향상시킵니다.

2. **캐시 효율성**: CPU 캐시는 데이터의 공간 지역성(spatial locality)을 활용합니다. 행 우선 순회는 연속된 메모리 주소에 접근하므로, 한 번 로드된 캐시 라인에서 여러 데이터를 읽을 수
   있습니다. 반면, 열 우선 순회는 메모리의 비연속적 접근을 초래하여 캐시 미스(cache miss)를 증가시킬 수 있습니다.

### 예제 코드

다음은 행 우선 순회와 열 우선 순회 간의 성능 차이를 보여주는 간단한 예제입니다:

```java
public class MatrixTraversal {
    private static final int SIZE = 1000;
    private static final int[][] matrix = new int[SIZE][SIZE];

    public static void main(String[] args) {
        long start, end;

        // 행 우선 순회
        start = System.nanoTime();
        rowMajorTraversal();
        end = System.nanoTime();
        System.out.println("Row-major traversal time: " + (end - start) + " ns");

        // 열 우선 순회
        start = System.nanoTime();
        columnMajorTraversal();
        end = System.nanoTime();
        System.out.println("Column-major traversal time: " + (end - start) + " ns");
    }

    private static void rowMajorTraversal() {
        for (int i = 0; i < SIZE; i++) {
            for (int j = 0; j < SIZE; j++) {
                matrix[i][j]++;
            }
        }
    }

    private static void columnMajorTraversal() {
        for (int j = 0; j < SIZE; j++) {
            for (int i = 0; i < SIZE; i++) {
                matrix[i][j]++;
            }
        }
    }
}
```

위 코드를 실행하면 행 우선 순회가 열 우선 순회보다 일반적으로 더 빠른 것을 확인할 수 있습니다. 이는 행 우선 순회가 메모리의 연속적인 위치에 접근하여 캐시 효율성을 극대화하기 때문입니다.

따라서, 자바에서 2차원 배열을 순회할 때 행 우선 순회를 사용하면 메모리 접근 패턴이 더 효율적이며, 성능 최적화에 도움이 됩니다.