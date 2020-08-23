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
reset명령은 soft, mixed, hard, merged, keep 총 5가지의 모드가 존재한다. 다음과 같은 이력을 가지고 있을 때, 각 모드 별 동작을 확인해 보자.
![git revert](/assets/img/posts/2020-08-18-git-rollback-git-reset-5mode-ready.png)

```bash
$ echo A > test
$ git add test
$ git commit -m "echo A" && git tag A
$ echo B > test
$ git commit -a -m "echo B" && git tag B
$ echo C > test
$ git commit -a -m "echo C" && git tag C
$ echo D > test
$ git commit -a -m "echo D" && git tag D
```

### soft
현재 HEAD가 D tag를 가르키고 있는 상태에서 B tag로 reset을 진행해보자.
```bash
$ cat test
D
$ git reset --soft B
$ cat test
D
```
reset 전과 후의 test file 내용이 똑같다.
```bash
$ git status
On branch master
Changes to be committed:
  (use "git restore --staged <file>..." to unstage)
	modified:   test
```
status를 조회하면 test 파일의 수정사항이 stage area에 올라와 있는 것을 알 수 있다.
```bash
$ git restore --staged
$ git checkout test
$ cat test
B
```
soft 모드의 경우 reset의 모드 중 가장 안전한 방법이다. HEAD 포인터가 가르키는 위치만 변경하고 working directory 작업 내용과 index는 유지하기 때문이다.

### mixed
mixed 모드로 reset을 진행해보자.
```bash
$ cat test
D
$ git reset --mixed B
$ cat test
D
```
soft 모드와 마찬가지로 reset 전과 후의 test파일이 똑같다.
```bash
$ git status
On branch master
Changes not staged for commit:
  (use "git add <file>..." to update what will be committed)
  (use "git restore <file>..." to discard changes in working directory)
	modified:   test
```
status를 조회하면 soft모드와 다른 것을 알 수 있다. soft모드의 경우 수정사항이 stage area에 올라와 있었는데 mixed 모드는 그렇지 않은 것을 확인할 수 있다.
```bash
$ git checkout test
$ cat test
B
```
git reset 명령 사용 시 mode에 대한 명시를 하지 않는다면 mixed 모드로 동작한다. mixed 모드의 경우 working directory 작업 내용만 유지한다.

### hard
hard 모드로 reset을 진행해보자.
```bash
$ cat test
D
$ git reset --hard B
$ cat test
B
$ git status
On branch master
nothing to commit, working tree clean
```
hard 모드는 HEAD 위치를 수정하고 index, working directory 작업 내용을 날려 버린다. 떄문에 가장 강력하고 위험한 방식이다. 다음과 같은 경우를 생각해보자.
```bash
$ cat test
D
$ echo E > test
$ cat test
E
```
위의 상태에서 soft, mixed 옵션으로 reset할 경우 test 파일에는 E라는 내용이 남아 있다. 하지만 hard모드의 경우 E라는 내용은 사라지고 더이상 E라는 내용을 복원할 수 있는 방법이 없다. 그렇기에 hard모드로의 reset은 신중해야 한다.

### merge

### keep

## revert

## 마무리