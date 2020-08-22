---
title: Git Rollback(reset, revert)
category: 
- DevOps
tags:
- git
summary: How to reset, revert, and return to previous states in Git
thumbnail: "/assets/img/thumbnail/2020-08-18-git-rollback-thumbnail.png"
---
Git을 사용하여 작업하다보면 이전 상태로 롤백하고 싶은 상황이 생기는데 reset, revert라는 명령어를 사용하여 해결할 수 있다. 두 명령어의 차이점과 사용방법에 대해 자세히 알아보자.

## reset vs revert
아래와 같은 이력을 가지고 있을 때, 8cc79c0 커밋으로 reset, revert할 때의 차이에 대해 알아보자.
![초기 상태](/assets/img/posts/2020-08-18-git-rollback-init-state.png)

### reset
reset을 사용할 경우 현재 가르키고 있던 HEAD 포인터를 바꿔버린다. 즉, 8cc79c0이후의 커밋 이력을 날리고 이전 상태로 돌아간다.
![git reset](/assets/img/posts/2020-08-18-git-rollback-git-reset.png)

### revert
revert를 사용할 경우 커밋이력을 유지한 상태로 새로운 커밋을 만든다. 8cc79c0 상태로 롤백했다는 새로운 커밋을 남기는 것이다.
![git revert](/assets/img/posts/2020-08-18-git-rollback-git-revert.png)

## reset
reset 명령어에 대해 자세히 알아보자. reset은 soft, mixed, hard, merged, keep 총 5가지의 모드가 존재한다.

### git reset --soft

## revert

## 마무리