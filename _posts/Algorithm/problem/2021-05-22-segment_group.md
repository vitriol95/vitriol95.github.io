---
title: 백준 선분 그룹 2162 with Python (BOJ)
date: 2021-05-21 22:20:00 +0900
categories: [Algorithm, problem]
tags: [algorithm,boj,problem,ccw,python3,union find,선분교차,백준]
math: true
---
# 문제 소개 - 선분 그룹
---
__문제출처__ : [백준 - 선분 그룹](https://www.acmicpc.net/problem/2162)

<img src="/assets/img/problems/boj2162.png">

# 풀이
---
## 접근과정

선분 교차에 대한 개념과 설명은 [이전글](https://vitriol95.github.io/posts/ccw/) 에서 많이 설명하였으므로 패스하겠다. 문제 푸는 과정은 간단하다. 이들이 서로 교차하는지(혹은 한점에서 만나는지)를 판단하고 이들을 같은 그룹으로 짝지어준다. 마지막으로 그룹의 수와 가장 큰 그룹에 선분 개수를 출력하면 된다.
<br>
여기서 '같은 그룹'으로 짝지어 준다는 것은 합집합을 진행하겠다는 의미이고, 이는 '유니온 파인드'를 이용하여 진행할 수 있다.

## 구현(python)
파이썬으로 구현한 코드는 다음과 같다.

```python
import sys
from collections import defaultdict
sys.setrecursionlimit(int(1e5))
input = sys.stdin.readline

def ccw(a,b,c):
    return a[0]*b[1]+b[0]*c[1]+c[0]*a[1]-(b[0]*a[1]+c[0]*b[1]+a[0]*c[1])

def find_parent(x):
    if parent[x] != x:
        parent[x] = find_parent(parent[x])
    return parent[x]

def union_parent(x,y):
    x = find_parent(x)
    y = find_parent(y)
    if x < y:
        parent[y] = x
    else:
        parent[x] = y

def check_twoline(l1,l2):
    if ccw(l1[0], l1[1], l2[0]) * ccw(l1[0], l1[1], l2[1]) <= 0 and
        ccw(l2[0], l2[1], l1[0]) * ccw(l2[0], l2[1], l1[1]) <= 0:
        if ccw(l1[0], l1[1], l2[0]) * ccw(l1[0], l1[1], l2[1]) == 0 and 
            ccw(l2[0], l2[1], l1[0]) * ccw(l2[0], l2[1], l1[1]) == 0:
            if max(l1[0][0], l1[1][0]) >= min(l2[0][0], l2[1][0]) and 
                    max(l2[0][0], l2[1][0]) >= min(l1[0][0], l1[1][0]) and 
                    max(l1[0][1], l1[1][1]) >= min(l2[0][1], l2[1][1]) and 
                    max(l2[0][1], l2[1][1]) >= min(l1[0][1], l1[1][1]):
                return True
        else:
            return True
    return False

n = int(input())
temp, parent = [], [i for i in range(n)]
line = []
for i in range(n):
    x1,y1,x2,y2 = map(int,input().split())
    temp.append([[x1,y1],[x2,y2]])
    for j in range(i):
        if check_twoline(temp[i],temp[j]):
            line.append([i,j])

for a,b in line:
    union_parent(a,b)

result = defaultdict(int)
for i in range(n):
    result[parent[i]]+=1
print(len(result.keys()))
print(max(result.values()))
```
> 첫번째 줄부터 차근 차근 살펴보도록 하자. 가장 먼저 선분이 교차하는지 (혹은 한점에서 만나는지)를 판별해주는 함수 ccw를 정의하고 그 뒤에 합집합(유니온파인드)를 진행하기 위한 2개의 함수 find_parent, union_parent를 정의해 주었다.
여기서 find_parent함수는 합집합을 진행하기전, 성분이 어느 그룹에 속해있는지를 알려주는 함수로써 경로압축 과정 `parent[x] = find_parent(parent[x])`를 넣어 일종의 memoization을 수행해 주고 있다.

그리고 가장 핵심 함수라고 할 수 있는 check_twoline이 있는데 이 함수는 선분 2개를 받아 ccw에 따른 선분의 교차 상태를 체크하고(이것에 대한 상세한 설명은 위에 링크해준 글에 존재한다) 교차하면 true 아니면 false를 뱉는다. 
<br>
39 ~ 44라인 까지는 성분들을 입력받아 이들이 교차하는지 브루트 - 포스 방식으로 그때 그때 체크해주고 교차하면 line이라는 곳에 넣어주어 이들을 union_parent해주면 된다. 마지막으로 parent(그룹)에 따른 개수와, 전체 그룹의 개수를 체크해주어야 하므로 dictionary 자료형을 사용하여 저장하였다.

## 한계? 
브루트 - 포스 방식으로 구현한 39~44 라인의 경우를 자세히 보면 새로운 선분이 추가될시, 기존에 있던 선분과 모든 상호작용(check_twoline)을 하게 되는데, 만약 temp에 50개 선분이 입력이 되어있고 새로운 선분이 추가될 경우 총 50번의 상호작용이 일어나게 된다. 여기서 선분의 개수는 3000개로 한정 되었기에 총 일어나는 상호작용의 개수는 $$3000*(3000+1)/2$$ 번이 될 것이다. 즉 적어도 O($$N^2$$)의 시간 복잡도를 갖기에, 선분의 개수가 크다면 적합한 방식은 아니다.