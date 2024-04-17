---
title: "[알고리즘] 누적합 알고리즘"
author: rdyjun
excerpt: "배열의 연속된 부분 합을 구하기에 최적화 알고리즘"

categories: [알고리즘]
tags: [java, algorithm, prefixsum]

toc: true
toc_sticky: true

date: 2024-04-17 21:20:00 +0800
last_modified_at: 2024-04-17 21:28:00 +0800

pin: false
---

누적합(prefix sum) 알고리즘은 작거나, 큰 배열에서 부분적으로 배열의 값을 업데이트 할 때 효율적으로 업데이트 할 수 있는 알고리즘이다.

카카오에서 출제한 파괴되지 않은 건물을 예시로 알아보겠다.

# 흐름

흐름을 간단하게 설명한다면, 임시 배열에 각 인덱스별 업데이트 할 최종 값을 담은 후 마지막에 이 임시 배열의 값을 원본 배열의 값과 합치는 방식이다.

### 1. 새로운 배열 선언

원본 배열과 동일한 크기의 배열을 만든다.<br>
그리고 해당 배열을 모두 0으로 초기화한다.

![alt text](/assets/img/prefix-sum-1.png)

### 2. 업데이트 해야 할 부분에 대해 적용

#### 업데이트 배열 설명

아래는 업데이트 할 부분에 대한 배열이다.<br>
각 배열의 원소는<br>

[0] type, 1일 때 degree를 빼며, 2일 때 degree를 더한다.
[1] row1, 첫 번째 행
[2] col1, 첫 번째 열
[3] row2, 두 번째 행
[4] col2, 두 번째 열
[5] degree, 연산할 값이다.

```
[[1,0,0,3,4,4],[1,2,0,2,3,2],[2,1,0,3,1,2],[1,0,1,3,3,1]]
```

> 배열의 첫 번째를 예시로, 0번 인덱스가 1이기 때문에 degree의 값인 4가 음수인 -4가 되며,
> 원본 배열에서 [row1][col1] 부터 [row2][col2] 까지 -4를 뺀다는 의미이다.

#### 배열 업데이트

새로운 배열의 `[row1][col1]` 부분에는 연산해야할 값`(-4)`를 넣고,<br>
`[row2 + 1][col]`, `[row1][co2 + 1]` 에는 연산해야할 값에 -1을 곱한 값`(4)`을 넣는다.<br>
마지막으로 `[row2 + 1][col2 + 1]` 에는 `[row1][col1]`에 넣은 값`(-4)`을 그대로 넣는다.<br>

하지만 사진에서 보는 것과 같이 `[row1][col1]`을 제외한 나머지 3개의 값은 밖으로 빠진다.<br>
걱정할 필요 없다. 배열에 넣지 않고 무시하면 된다.

![alt text](/assets/img/prefix-sum-2.png)

#### 궁금한점?

이게 무슨 원리인가?<br>
이따 최종적으로 연산을 한 번 더 하겠지만, 간단히 설명하자면<br>
`[0][0]` 부터 위의 값과 좌측의 값을 더하고, 대각선 좌측 위의 값을 빼보자.<br>
그렇다면 아래와 같은 배열이 나오게 된다.
![alt text](/assets/img/prefix-sum-3.png)

아마 누구나 위 사진을 보면 바로 이해할 것이라고 생각한다.

물론 배열 밖의 값들은 배열에 적용되지 않지만, 설명을 위해 넣었다.<br>
정확히 우리가 빼고자 했던 `[0][0]` 부터 `[3][4]` 까지 -4로 적용했다.<br>
또한 그 바깥은 0으로 다시 돌아온다.<br>
자세한 설명은 계속해서 보자.

#### 2번째 배열 업데이트

[1,2,0,2,3,2]

![alt text](/assets/img/prefix-sum-4.png)

#### 3번째 배열 업데이트

[2,1,0,3,1,2]

![alt text](/assets/img/prefix-sum-5.png)

#### 4번째 배열 업데이트

[1,0,1,3,3,1]

![alt text](/assets/img/prefix-sum-6.png)

### 부분합 계산

드디어 모든 부분 배열에 대한 업데이트 배열을 만들게 되었다.<br>
이제 첫 번째 배열 업데이트 때 했던 계산을 해보자.<br>

`[0][0]`부터 배열의 끝`([3][4])`까지 아래 연산을 진행할 것이다.

```java
arr[r][c] += arr[r - 1][c] + arr[r][c - 1] - arr[r - 1][c - 1]
```

그렇다면 아래와 같은 결과가 나오게 된다.

![alt text](/assets/img/prefix-sum-7.png)

### 원본 배열과 합

본 문제에서는 결과 값이 0보다 큰 인덱스의 수만 알아내면 되기 때문에 직접 배열에 더하진 않았다.<br>
각 문제에 맞게 적용해서 사용하자.

# 최종 코드

```java
import java.util.Arrays;

class Solution {

    private static final Integer TYPE_INDEX = 0;

    private static final Integer R1_INDEX = 1;

    private static final Integer C1_INDEX = 2;

    private static final Integer R2_INDEX = 3;

    private static final Integer C2_INDEX = 4;

    private static final Integer DEGREE_INDEX = 5;

    public int solution(int[][] board, int[][] skill) {
        int boardHeight = board.length;
        int boardWidth = board[0].length;

        int[][] prefixSumArray = getPrefixArray(boardHeight, boardWidth, skill);

        calculatePrefixSum(boardHeight, boardWidth, prefixSumArray);

        int answer = 0;

        for (int row = 0; row < boardHeight; row++) {
            for (int col = 0; col < boardWidth; col++) {
                if (prefixSumArray[row][col] + board[row][col] > 0) {
                    answer++;
                }
            }
        }

        return answer;
    }

    private void calculatePrefixSum(int height, int width, int[][] prefix) {
        for (int row = 0; row <= height; row++) {
            for (int col = 0; col <= width; col++) {
                int a = 0;
                int b = 0;
                int c = 0;

                if (row - 1 >= 0) {
                    a = prefix[row - 1][col];
                }

                if (col - 1 >= 0) {
                    b = prefix[row][col - 1];
                }

                if (row - 1 >= 0 && col - 1 >= 0) {
                    c = prefix[row - 1][col - 1];
                }

                prefix[row][col] += a + b - c;
            }
        }
    }

    private int[][] getPrefixArray(int height, int width, int[][] skill) {
        int[][] prefixSumArray = new int[height + 1][width + 1];

        int type, r1, c1, r2, c2, degree;

        for (int[] eachSkill : skill) {
            type = eachSkill[TYPE_INDEX];
            r1 = eachSkill[R1_INDEX];
            c1 = eachSkill[C1_INDEX];
            r2 = eachSkill[R2_INDEX];
            c2 = eachSkill[C2_INDEX];
            degree = eachSkill[DEGREE_INDEX];

            if (eachSkill[0] == 1) {
                degree *= -1;
            }

            prefixSumArray[r1][c1] += degree;
            prefixSumArray[r2 + 1][c1] -= degree;
            prefixSumArray[r1][c2 + 1] -= degree;
            prefixSumArray[r2 + 1][c2 + 1] += degree;
        }

        return prefixSumArray;
    }

}

```

# 결론

처음에는 다른 블로그에서 설명해준 누적합을 통해 학습했다.<br>
그 블로그들에서는 행 계산 따로, 열 계산 따로 하는 코드로 확인했다.<br>
하지만 더 줄일 수 있을 것 같아 알아본 결과... 한 번에 할 수 있었다.<br>
하지만 이로 인해 궁금증이 생겼었다.

##### 어떻게해서 앞에서와 같은 식이 적용되는 것일까?

```bash
arr[r][c] += arr[r - 1][c] + arr[r][c - 1] - arr[r - 1][c - 1]
```

물론 앞에서도 설명했다. 하지만 조금더 길게 설명해보고자 한다.

우리는 `arr[r1][c1]`에 연산 해야 할 값을 넣었고, `arr[r2 + 1][c1]` 과 `arr[r1][c2 + 1]` 에 값에 -1을 곱한 값을 넣었다.<br>

이 행동의 의미는 "해당 위치를 넘어갔을 때 다시 0으로 돌아와라" 라는 것을 의미한다.<br>
즉, 범위를 넘어갔을 때 아무 값도 계산하지 않도록 바깥 범위에 반대의 값`(* -1)`을 넣은 것이다.<br>

최종적으로 `arr[r2 + 1][c2 + 1]`을 들어왔을 때 바로 위와 왼쪽인 `arr[r2 + 1][c2]`와 `arr[r2][c2 + 1]`는 확실하게 0이고, `arr[r2][c2]`는 확실하게 -4기 때문에 `0 + 0 - (-4) + ?`와 같은 식에서 `?`가 `- (-4)`와 반대인 `-4`가 들어가야했던 것이다.<br>

![alt text](/assets/img/prefix-sum-8.png)

결론은 이러한 부분 업데이트 방식은 우리가 원하는 부분만 정확히 수정하지만, 더 효율적으로 수정하는 좋은 알고리즘이라고 볼 수 있다. 처음 이 알고리즘 문제를 마주했을 때, 어떻게 풀지 막막했다. 풀이법을 알아내고 생각보다 쉬워 많이 당황했다...
