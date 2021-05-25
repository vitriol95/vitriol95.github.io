---
title: 백준 적록색약 10026 with Python
date: 2021-05-24 15:00:00 +0900
categories: [Algorithm, problem]
tags: [algorithm,boj,problem,dfs,bfs,그래프,python3,백준]
---

# 문제소개
---
__문제출처__ : [백준 - 적록색약](https://www.acmicpc.net/problem/10026)

<img src="/assets/img/problems/boj10026.PNG">

# 풀이
---
## 접근 과정

간단한 bfs / dfs 문제중 하나이다. 두가지를 구현을 해야하는데 적록색약이 아닌경우와 적록색약일 경우를 각각 bfs / dfs를 이용해 구현하면 된다.

<br>
방문했는지의 여부를 나타내는 visited를 초기화 시켜놓고, 이중 for문으로 그래프 하나하나를 돌면서 미방문한 위치가 나올 시 dfs로 인접한 타일의 방문처리를 해주고, 이 때마다 카운트를 하나씩 더해주면 된다. 

<br>
여기서는 dfs로 구현해 보았다.

## 구현 (python)
```python
import sys
sys.setrecursionlimit(int(1e6))
def dfs(x,y,mode=0):

    if mode==0:
        col = graph[x][y]
        visited[x][y] = True
        for i in range(4):
            nx = x +dx[i]
            ny = y +dy[i]
            if nx < 0 or nx >=n or ny < 0 or ny >= n:
                continue
            if visited[nx][ny] or graph[nx][ny]!=col:
                continue
            dfs(nx,ny,mode)

    # 적록 색약인 경우
    else:
        col = graph[x][y]
        visited2[x][y] = True
        for i in range(4):
            nx = x +dx[i]
            ny = y +dy[i]
            if nx < 0 or nx >= n or ny < 0 or ny >= n:
                continue
            if visited2[nx][ny]:
                continue
            if col == 'R' or col == 'G':
                if graph[nx][ny] == 'R' or graph[nx][ny] == 'G':
                    dfs(nx,ny,mode)
            else:
                if graph[nx][ny] != col:
                    continue
                dfs(nx,ny,mode)

n = int(input())
graph = []
for _ in range(n):
    graph.append(list(input()))

visited = [ [False for _ in range(n)] for _ in range(n)]
visited2 = [ [False for _ in range(n)] for _ in range(n)]
count1, count2 = 0, 0
dx, dy = [0,0,-1,1], [-1,1,0,0]

for i in range(n):
    for j in range(n):
        if not visited[i][j]:
            dfs(i,j)
            count1+=1
        if not visited2[i][j]:
            dfs(i,j,1)
            count2+=1
print(count1,count2)
```
> 코드를 차근차근 살펴보자 dfs 메서드의 경우 mode라는 매개변수를 나누어 적록색약인 경우(1)와 그렇지 않은 경우(0)를 나누어서 처리하였다. 두 mode의 다른점이라면 28~30 라인에 해당한다. 타일값이 R과 G인 경우 이를 동일시 하여 처리하는 것이다.

위 코드를 좀 더 간결하게 설명해 보자면

1. 방문처리 및 카운트를 2가지(적록색약x, 적록색약) 만들기 (visited,visited2) 
2. 적록색약이 X(mode=0)인 경우에는 제일 처음 들어온 color 값을 저장하고, 이와 같은 경우만 방문처리를 해주고 나머지는 패스
3. 적록색약(mode=1)인 경우에는 R과 G를 함께 처리해주고 B인경우 B만 방문 처리해준다.

> 46~53 부분이 실제 동작하는 모습에 해당한다. 그래프의 칸 하나씩을 돌며 방문되지 않았을 경우 그 칸의 색상을 기준으로 방문처리를 해주는 것이다.

## 한계 ?
일단 mode=0, mode=1에 겹치는 부분들이 많아 이를 없애는 작업을 더 수행할 수 있다. 그리고 n의 수가 큰 경우에는 46~53 줄 처럼 이중 for문으로 진행할 경우 시간초과를 불러올 수 있다.