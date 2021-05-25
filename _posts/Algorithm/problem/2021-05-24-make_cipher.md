---
title: 백준 암호 만들기 1759 with Python
date: 2021-05-24 10:00:00 +0900
categories: [Algorithm, problem]
tags: [algorithm,boj,problem,조합,브루트 포스,백트래킹,python3,백준]
---

# 문제소개
---
__문제출처__ : [백준 - 암호 만들기](https://www.acmicpc.net/problem/1759)

<img src="/assets/img/problems/boj1759.PNG">

# 풀이
---
## 접근 과정

<br>
조합을 이용한 문제이다. 하지만 다른 조합 문제와는 다르게 특별한 점이 있는데 바로 자음과 모음이 각각 1개와 2개 이상 필수적으로 들어가야 하는 것이다. 마치 조금 꼬아낸 경우의 수 문제와도 같다. 그리고 한가지 더, 오름차순으로 정렬해야 한다는 점이 존재한다. 

<br>
따라서 모음과 자음 조건을 종료조건에 추가해 주기만 하면 된다. 파이썬 표준 라이브러리인 itertools.combination을 이용해도 되지만, 여기서는 직접 브루트 포스로 조합을 구현해 진행했다.

## 구현(python)
```python
import sys
sys.setrecursionlimit(int(1e5))
input = sys.stdin.readline

def dfs(temp,consonant,vowel,n):

    if n == l:
        if consonant >=2 and vowel >=1:
            result.append(temp)
            return

    for i in range(len(char)):

        if visited[i]:
            continue
        if temp and temp[-1] > char[i]:
            continue
        if char[i] in ['a','e','i','o','u']:
            visited[i] = True
            dfs(temp+[char[i]],consonant,vowel+1,n+1)
            visited[i] = False
        else:
            visited[i] = True
            dfs(temp+[char[i]],consonant+1,vowel,n+1)
            visited[i] = False

l,c = map(int,input().split())
char = list(input().rstrip().split())
visited = [False for _ in range(c)]
result = []
dfs([],0,0,0)
result.sort()
for i in result:
    print("".join(i))
```
> 18~25 줄이 가장 핵심이라고 볼 수 있다. 자음일 경우와 모음일 경우를 나누어서 함수의 매개변수에 자음(consonant)과 모음(vowel)의 개수를 넣어준다. 그리고 배열에 모인 원소의 개수(마지막 매개변수)가 주어진 l값과 동일해지면 종료한다. 이때, 자음과 모음 개수도 체크한다. (7~10줄)