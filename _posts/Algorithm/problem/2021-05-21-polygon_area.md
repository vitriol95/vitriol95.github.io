---
title: 백준 다각형의 면적 2166 with Python (BOJ)
date: 2021-05-21 22:00:00 +0900
categories: [Algorithm, problem]
tags: [algorithm,boj,problem,다각형의 면적,python3,백준]
---

# 문제 소개
---
__문제 출처__ : [백준 - 다각형의 면적](https://www.acmicpc.net/problem/2166)
<br>
<img src="/assets/img/problems/boj2166.PNG">

# 풀이 
---
## 접근 과정

대학 수학이나, 공업 수학에서 배우는 그린정리의 특수한 형태라고 볼 수 있다. 2차원에서 좌표 값이 주어질 때(N각형이며, 오목 볼록과는 상관 없이 사용할 수 있다.) 다각형의 면적을 구하는 방법으로 __'신발끈 정리'__ 라고도 불린다. 
<br>
원리는 정말 간단하다. 단순히 말하면. 각각 모서리 마다 원점(0,0)을 기준으로 하는 삼각형의 넓이를 계산하여, 평행 사변형의 넓이와 같은 벡터곱(cross product)들을 모두 합쳐 2로 나누면 끝이다. 만약 원점 기준으로 모서리들이 반시계 방향으로 돈다면, 양의 면적이 더해지게 되고 반시계 방향으로 돈다면 음의 면적이 더해지게 되는 것이다. 이는 CCW의 개념과 비슷하다. 하지만 여기서는 넓이(양의 값)만 알면 되므로 아래와 같이 절대값 처리를 해주면 끝이다.

<img src="/assets/img/problems/boj2166-2.PNG">

_자세한 설명은 [위키피디아](https://ko.wikipedia.org/wiki/%EC%8B%A0%EB%B0%9C%EB%81%88_%EA%B3%B5%EC%8B%9D) 혹은 [mathworld](https://mathworld.wolfram.com/PolygonArea.html)을 참고하면 좋다._

## 구현(Python)
```python
import sys
input = sys.stdin.readline
n = int(input())
x, y = [], []
answer = 0
for _ in range(n):
    a, b = map(int,input().split())
    x.append(a)
    y.append(b)
x, y = x + [x[0]], y + [y[0]]

for i in range(n):
    answer += (x[i]*y[i+1])-(x[i+1]*y[i])
print(round(abs(answer)/2,1))
```
> 앞선 식(A)의 구현부 밖에 쓴것이 없다. 각각의 행렬식을 모두 더해주는 부분(13줄)이 핵심이라 볼 수 있다. 단순한 표현에 비해 이론이 굉장히 어려워서 (사실 아직도 그린정리는 잘 이해하지 못했다..) 정답률이 낮은 것이 아닌가 싶다.