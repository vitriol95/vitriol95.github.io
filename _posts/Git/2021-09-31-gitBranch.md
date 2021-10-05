---
title: git 브랜치, merge 전략
date: 2021-09-31 11:00:00 +0900
categories: [Git]
tags: [git,branch,merge]
---

## Git Branch + Merge

#### 브랜치이름
- 평소에 브랜치 이름을 이슈이름을 따서 feature/{issue-name} 지었었는데, 이 방식 보다는 feature/{issue-number}로 가는 것이 좋다. 깃허브와 자동 연결된다.!
- 커밋 메시지를 달 때도, [#{issue-number}]를 맨 앞에 붙혀주는 것이 좋다. 이 역시 자동 연결을 지원한다.

#### PR 종료시 제공하는 3가지 merge 방식
1. Create a merge Commit
  - 디폴트 값으로 나와있는데, 선택하면 베이스 브랜치에 모든 커밋이 달라 붙게 된다.
  - 내가 git flow finish 명령어를 사용했을 때 진행되는 것과 비슷하다.
  - merge 커밋 이력이 남게 된다.
2. Squash and merge
  - feature 브랜치에 커밋된 여러 변경사항들(커밋들)을 1개로 압축하여 베이스 브랜치에 넣어 준다.
  - 자잘자잘한 변경사항들만 존재할 때 한번에 넣는 방식으로 좋을 것 같다. (피처일이 아주 작을 경우) 
3. Rebase and merge
  - feature 브랜치에 커밋된 변경사항들을 베이스 브랜치에 넣어주고, 이를 최상단에 위치시켜 준다.
  - 이후에 베이스로 왔을 때, 그 변경사항들이 모두 적용되어있다 (untracked된것이 없어진다.)
  - 2번째 방식과 반대 상황일 때 쓰면 좋을 것 같다. 

## Commit Message

앞선 커밋 네이밍을 다르게 진행하는 방식도 있다.

- 가장 먼저, 헤더에는 커밋의 성격을 나타내면 된다. ()안에 대상을 명시해 주는 것도 좋다.
  - feat: 새기능 / fix: 버그 수정 / build: 빌드 관련 / chore: 자잘한 것
  - ci: ci관련 / docs: 문서 수정 / style: 코드 스타일및 포맷 / refactor: 코드 리팩토링 / test: 테스트 수정
  - 이후, 간단한 설명글을 쓴다
- 마찬가지로 참조하는 이슈를 넣어주면 좋다.
> 이들로 부터 예를 들어보면, [#issue-number]feat(admin-service): Add search function 처럼 헤더를 작성하는 것이 좋을 것 같다.

- 제목은 최대한 50글자 안으로 짓고, 마침표를 사용하지 말며 명령문으로 작성해야한다. 또한 '무엇'과 '왜'가 들어가야 좋다.
- 본문의 경우에는 앞서 제목에 설명한 내용의 세부 설명을 넣어준다.

## 참고자료
- [https://beomseok95.tistory.com/328](https://beomseok95.tistory.com/328)
- [https://cornswrold.tistory.com/535](https://cornswrold.tistory.com/535)