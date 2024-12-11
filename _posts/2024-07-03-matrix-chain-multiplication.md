---
title: "[코딩테스트] 연속된 행렬의 최소 곱셈 수"
author: rdyjun
excerpt: "연속된 행렬을 모두 곱하기 위해 필요한 최소 곱셈 수 구하기"

categories: [coding-test]
tags: [dynamicprogramming, dynamic, matrix]

toc: true
toc_sticky: true

date: 2024-07-03 23:45:00 +0900
last_modified_at: 2024-06-16 09:25:00 +0900

pin: false
published: false
---

# ❓ 문제

- 최적의 행렬 곱셈
- Programmers
- Level. 3
- [https://school.programmers.co.kr/learn/courses/30/lessons/12942](https://school.programmers.co.kr/learn/courses/30/lessons/12942)

## 📃 문제 설명

크기가 a by b인 행렬과 크기가 b by c 인 행렬이 있을 때, 두 행렬을 곱하기 위해서는 총 a x b x c 번 곱셈해야합니다.
예를 들어서 크기가 5 by 3인 행렬과 크기가 3 by 2인 행렬을 곱할때는 총 5 x 3 x 2 = 30번의 곱하기 연산을 해야 합니다.

행렬이 2개일 때는 연산 횟수가 일정 하지만, 행렬의 개수가 3개 이상일 때는 연산의 순서에 따라서 곱하기 연산의 횟수가 바뀔 수 있습니다. 예를 들어서 크기가 5 by 3인 행렬 A, 크기가 3 by 10인 행렬 B, 크기가 10 by 6인 행렬 C가 있을 때, 순서대로 A와 B를 먼저 곱하고, 그 결과에 C를 곱하면 A와 B행렬을 곱할 때 150번을, AB 에 C를 곱할 때 300번을 연산을 해서 총 450번의 곱하기 연산을 합니다. 하지만, B와 C를 먼저 곱한 다음 A 와 BC를 곱하면 180 + 90 = 270번 만에 연산이 끝납니다.

각 행렬의 크기 matrix_sizes 가 매개변수로 주어 질 때, 모든 행렬을 곱하기 위한 최소 곱셈 연산의 수를 return하는 solution 함수를 완성해 주세요.

## 🚫 제한사항

- 행렬의 개수는 3이상 200이하의 자연수입니다.
- 각 행렬의 행과 열의 크기는 200이하의 자연수 입니다.
- 계산을 할 수 없는 행렬은 입력으로 주어지지 않습니다.

## 🖥️ 입출력 예

| matrix_sizes              | result |
| ------------------------- | ------ |
| [[5,3], [3, 10], [10, 6]] | 270    |

## 🚀 문제 접근 방식

### 📜 행렬곱 연산 방식

행렬의 곱은 `A x B`, `B x C`의 행렬에 대해서만 이루어진다.<br>
즉, 첫 번째 행렬의 `열의 수`와 두 번째 행렬의 `행의 수`가 같아야 한다.<br>

문제에서는 `[5, 3]`, `[3, 10]` 과 같이 위 조건을 만족하는 행렬을 입력해주고 있다.<br>

![alt text](/assets/img/matrix-multiple-2.png)

여기서 곱셈을 하는 횟수는 겹치지 않는 두 부분을 곱하고, 겹치는 부분을 한 번만 곱하는 것이다.<br>
즉, 겹치지 않는 `5`와 `10`을 곱하고, 겹치는 부분인 `3`을 한 번만 곱한 `150`이 된다.

### 🗂️ 입력된 행렬의 구조

문제에서는 `"계산할 수 없는 행렬은 입력으로 주어지지 않는다"` 라고 명시하였다.<br>
이 부분에서 입력된 행렬들은 순서대로 곱셈이 가능하다고 유추해볼 수 있다.

![alt text](/assets/img/matrix-multiple-1.png)

### 📈 행렬곱 패턴

위에서 설명했듯, 두 행렬의 곱을 수행하면서 겹치지 않는 크기의 두 너비가 행렬의 크기가 된다.<br>
그렇다면 3개의 행렬을 곱해보겠다.<br>

1. 예시를 위해 순서대로 `[5, 3]`과 `[3, 10]`을 곱한다.
2. 크기가 같은 3을 제외한 `[5, 10]`의 크기로 행렬이 나타나게 된다.
3. 여기에 `[10, 6]`을 곱하면 `[5, 6]` 크기의 행렬이 나타나게 된다.

여기서 또 한 가지 알 수 있는 사실은 세 개의 행렬을 모두 곱했을 때,<br>
결과적으로 가장 왼쪽에 있는 행렬의 행의 크기와,<br>
가장 오른쪽에 있는 행렬의 열의 크기가 최종 행렬의 크기가 된다.<br>

이 부분에 있어서 이후에 계속해서 다루겠다.

![alt text](/assets/img/matrix-multiple-3.png)

### 🚨 집중해야할 문제

위에서는 순서대로 `[5, 3]`과 `[3, 10]`을 곱했다.<br>
이에따른 총 곱셈 연산 수를 계산해보겠다.<br>
`5 * 3 * 10 = 150`<br>
`5 * 10 * 6 = 300`<br>

`150`과 `300`을 더한 `450`이 나오게 된다.<br>
하지만 곱셈의 순서를 바꿔보면 어떻게 나올까?<br>
2번째 행렬과 3번째 행렬을 곱해보겠다.<br>
`3 * 10 * 6 = 180`<br>
`5 * 3 * 6 = 90`<br>

결과적으로 `270`이 나와, 앞서 계산한 `450`보다 작은 값이 나오게 된다.<br>
이렇게 경우의 수를 구하여 최소한의 곱셈을 수행하는 순서를 찾아야 한다.

### 🥇 최적의 행렬 곱 순서?

행렬곱의 최소 곱셈 수를 찾기 위해서는, 각 순서를 실행해보는 방법이 있다.<br>
`dfs` 방식으로도 문제를 해결할 수 있겠으나, 제한된 메모리에서는 해결이 불가능할 것이다.<br>
그렇기 때문에 각 행렬에 대한 곱셈 결과를 저장하고, 비용을 따져가며 결과를 도출해야한다.<br>

![alt text](/assets/img/matrix-multiple-4.png)

위 그림에서는 `1번`, `2번`의 행렬에 대한 곱셈이 `150`번 이루어지고,<br>
`2번`, `3번` 행렬에 대한 곱셈이 `180`번 이루어지는 것을 볼 수 있다.<br>
이 값에 대한 정보를 저장해두고, 나머지 행렬에 대한 곱셈 값을 구하면 결과를 비교할 수 있다.<br>

[1, 2, 3]순서의 행렬곱은 `450`, [2, 3, 1]순서의 행렬곱은 `270`<br>
결론적으로 1번부터 3번까지의 최소 행렬곱은 `270`이 된다.<br>

여기서 나는 한 가지 생각이 들었다.<br>
<b>그럼 더 많은 행렬은 어떻게 처리하지?</b><br>
그렇다. 이렇게 영역을 1개부터 행렬의 개수만큼 늘려가며 연산해보면 된다.

![alt text](/assets/img/matrix-multiple-5.png)

위 그림은 최종적으로 6개의 행렬곱의 최소 곱셈 비용을 찾는 경우의 수 이다.<br>

> 여기서 6개의 행렬곱의 최소 곱셈 비용을 찾는다는 의미는,<br>
> 1개부터 5개까지의 행렬곱의 최소 곱셈 비용을 알고있다는 의미이다.

이런식으로 경우의 수를 비교해가며 최소 곱셈 비용을 최종적으로 찾을 수 있다.

### 🔥 점화식

위 내용을 토대로 추상적인 흐름을 이해할 수 있었다.<br>
그렇다고 하면, 어떻게 각 행렬에 대한 최소 곱셈 비용을 저장해야할까?<br>
2차원 배열을 사용하여 각 행렬 곱에 대한 최소 곱셈 비용을 저장했다.<br>

```java
int[][] dp = new int[matrix.length][matrix.length];
```

이 2차원 배열은 `행`이 `시작지점`이고, `열`이 `도착지점`이라고 생각하면 된다.<br>
즉, `dp[a][b]`라면 a부터 b까지의 최소 행렬 곱셈 비용을 저장하는 공간이라는 뜻이다.<br>

그렇다면 먼저, `a`와 `b`가 같은 상황일 때, 즉 행렬이 한 개만 있는 경우는<br>
행렬곱을 사용하지 않기 때문에 0이 된다.

![alt text](/assets/img/matrix-multiple-6.png)
바로 이어서 두 개의 행렬곱을 수행하는 경우를 알아보겠다.<br>
a가 1이고, b가 2일 때 `1행렬`부터 `2행렬`까지의 곱셈 비용이다.<br>
1번 행렬과 2번 행렬의 행렬곱 곱셈 비용은 `5 * 3 * 10`인 `150`이다.

![alt text](/assets/img/matrix-multiple-7.png)

다음으로 a가 2이고, b가 3일 때 `2행렬`부터 `3행렬`까지의 곱셈 비용이다.<br>
2번 행렬과 3번 행렬의 행렬곱 곱셈 비용은 `3 * 10 * 6`인 `180`이다.<br>

![alt text](/assets/img/matrix-multiple-8.png)

여기까지 2개씩 짝지은 행렬의 곱셈 비용을 저장했다.<br>
이제부터 3개씩 짝지은 행렬의 곱셈 비용을 찾아볼 것이다.<br>
3개씩 짝지은 행렬의 곱셈은 `1행렬`부터 `3행렬`까지의 행렬곱밖에 없다.<br>

여기서 생각할 수 있는 것은 경우의 수를 대입해보는 것이다.<br>
우리는 `1번~2번` 행렬의 곱셈 비용, `2번~3번` 행렬의 곱셈 비용을 알고있다.<br>

![alt text](/assets/img/matrix-multiple-9.png)

위 이미지와 같이 `1번~2번` 행렬 크기에 `3번`행렬을 곱하고 최종 결과를 얻으면 되고,<br>
마찬가지로 `2번~3번` 행렬 크기에 `1번`행렬을 곱하고 최종 결과만 얻으면 된다.<br>
이렇게 나온 결과에 가장 최소 값인 비용이 `1번~3번` 행렬의 최소 곱셈 수 이다.<br>

![alt text](/assets/img/matrix-multiple-10.png)

여기서 한 가지 궁금증이 생길수도 있을 것 같다.<br>
그러면 `1번~2번` 행렬에 3번 행렬을 어떻게 곱하느냐.

위 `행렬곱 패턴` 부분에서 다룬 내용이다.<br>
행렬의 곱의 결과로 나온 크기는 `가장 왼쪽 행렬의 행` 크기와 `가장 오른쪽 행렬의 열` 크기가 된다.<br>

```java
int row = dp[left][0];
int col = dp[right][1];
```

그렇기 때문에 1번 행렬부터 2번 행렬까지의 행렬곱 이후 크기는<br>
1번 행렬의 행과 2번 행렬의 열이다.<br>

```java
dp[left][right] = new int[]{row, col};
```

결론적으로 아래와 같은 식이 작성될 수 있다.<br>

```java
int multiple = matrix[a][0] * matrix[k][1] * matrix[b][1]
```

- `matrix[a][0]`는 가장 왼쪽에 위치한 행렬이다.
- `matrix[k][1]`는 가운데 위치한 행렬이다.
- `matrix[b][1]`는 가장 오른쪽에 위치한 행렬이다.

이렇게 나온 두 연속된 행렬의 곱셈 값에 각 비용을 더하면 된다.

```java
multiple += dp[a][k] + dp[k + 1][b];
```

- `dp[a][k]`는 a부터 k까지의 행렬곱 곱셈 비용이다.
- `dp[k + 1][b]`는 k + 1부터 b까지의 행렬곱 곱셈 비용이다.

결과적으로 각 경우의 수를 대입할 것이기 때문에,<br>
`Math.min()`으로 값을 비교하여 업데이트해주면 된다.

```java
dp[a][b] = Math.min(dp[a][b], dp[a][k] + dp[k + 1][b] + (dp[k][]))
```

# 🔖 최종 해결 코드

```java
import java.util.Arrays;

class Solution {

    private int matrixCount;
    private int[][] cost;
    private int[][] matrix_sizes;

    public int solution(int[][] matrix_sizes) {
        this.matrixCount = matrix_sizes.length;
        this.cost = initializeArray(matrixCount);
        this.matrix_sizes = matrix_sizes;

        findMinCost();

        return this.cost[0][this.matrixCount - 1];
    }

    private void findMinCost() {
        // 연산할 행렬의 수 0일 때는 1개를 의미
        for (int multipleMatrixCount = 0; multipleMatrixCount < this.matrixCount; multipleMatrixCount++) {
            // 연산을 시작할 행렬의 인덱스
            rotateMatrix(multipleMatrixCount);
        }
    }

    //** 각 행렬을 순회하여 최소 값 연산 */
    private void rotateMatrix(int multipleMatrixCount) {
        for (int index = 0; index < this.matrixCount; index++) {
            int start = index; // 시작 위치
            int end = index + multipleMatrixCount; // 마지막 위치

            if (end >= matrixCount) {
                continue;
            }

            if (start == end) {
                this.cost[start][end] = 0;
                continue;
            }

            particionCost(start, end);
        }
    }

    /** 시작위치와 마지막 위치를 입력받아 배열을 둘로 나누어 연산해서 가장 적은 비용의 곱셈 수 찾기 */
    private void particionCost(int start, int end) {
        // 왼쪽과 오른쪽을 나누는 중간을 이동시키며 비교
        for (int middle = start; middle < end; middle++) {
            int left = this.cost[start][middle];
            int right = this.cost[middle + 1][end];
            // 왼쪽과 오른 쪽을 이을 때 필요한 곱셈 수
            int multiple = this.matrix_sizes[start][0] * this.matrix_sizes[middle][1] * this.matrix_sizes[end][1];

            // start부터 end까지 곱셈 수는 left + right + multiple
            this.cost[start][end] = Math.min(this.cost[start][end], left + right + multiple);
        }
    }

    private int[][] initializeArray(int matrixCount) {
        int[][] array = new int[matrixCount][matrixCount];
        for (int i = 0; i < matrixCount; i++) {
            Arrays.fill(array[i], Integer.MAX_VALUE);
        }

        return array;
    }
}

```

### 💎 회고

본 문제를 처음 만났을 때는 작년(2023년) 12월이었다.<br>
이 문제를 도통 어떻게 풀어야할지 감이 오지 않아,<br>
힌트를 보고 한동안 풀지 않겠다고 하다가, 이번에 풀게 되었다.

나는 이렇게 제한이 없었다고 가정할 때, DFS로 풀이가 가능한 문제를 마주하면 고민에 빠지게 된다.<br>
그 이유는 머리로는 점화식을 세우고, 동적 계획법으로 접근해야 하는데,<br>
아무 생각도 나지 않는다는 것이다...

오늘은 이전의 힌트를 본 기억을 살려 문제를 접근해보려고 하였고,<br>
결과적으로 점화식을 세워 문제를 풀 수 있었다!

앞으로 이런 문제들을 많이 접하고, 동적 계획법에 익숙해져야겠다.<br>

단순하지만, 어려운 문제
