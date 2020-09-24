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
아래와 같은 이력을 가지고 있는 상태에서 8cc79c0 커밋으로 reset, revert할 때의 차이에 대해 알아보자.
![초기 상태](/assets/img/posts/2020-08-18-git-rollback-init-state.png)

### reset
reset을 사용할 경우 현재 가르키고 있던 HEAD 포인터를 8cc79c0로 바꿔버린다. 즉, 8cc79c0이후의 커밋 이력을 날리고 이전 상태로 돌아간다.
![git reset](/assets/img/posts/2020-08-18-git-rollback-git-reset.png)

### revert
revert를 사용할 경우 커밋이력을 유지한 상태로 8cc79c0커밋에 대한 내용을 취소하고 revert를 실행했다는 새로운 커밋을 남긴다. 8cc79c0커밋 상태로 돌아가는 것이 아닌 8cc79c0커밋의 내용을 취소하고, 취소했다는 커밋을 남기는 것이다.
![git revert](/assets/img/posts/2020-08-18-git-rollback-git-revert.png)

## reset
reset명령은 soft, mixed, hard, merged, keep 총 5가지의 모드가 존재한다. 다음과 같은 이력을 가지고 있을 때, 각 모드 별 동작을 확인해 보자.
![git reset](/assets/img/posts/2020-08-18-git-rollback-git-reset-5mode-ready.png)
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
현재 HEAD가 D에 있는 상태에서 B로 reset을 진행해보자.
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
soft 모드의 경우 reset의 모드 중 가장 안전한 방법이다. HEAD 포인터의 위치만 변경하고 working directory 작업 내용과 index는 유지하기 때문이다.

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

### keep
keep모드는 hard모드와 비슷하지만 보다 안전한 방법이다. keep 모드로 reset을 진행해 보자.
```bash
$ cat test
D
$ git reset --keep B
$ cat test
B
$ git status
On branch master
nothing to commit, working tree clean
```
keep 모드 역시 hard와 마찬가지로 HEAD 위치를 수정하고 index, working directory을 날려 버린다. 하지만 변경사항이 있는 경우 reset을 진행하지 않는다.
```bash
$ cat test
D
$ echo E > test
$ cat test
E
$ git reset --keep B
error: Entry 'test' not uptodate. Cannot merge.
fatal: Could not reset index file to revision 'B'.
$ cat test
E
$ git reset --hard B
$ cat test
B
```
hard모드로 reset을 진행할 경우 test 파일의 E라는 내용을 복원할 방법이 없다. 반면 keep모드의 경우 이와같은 reset 자체를 허용하지 않는다. 때문에 hard모드의 reset보다 안전한 방법이라 할 수 있다.

### merge
merge모드는 동작 방식이 keep모드와 비슷하다.
```bash
$ cat test
D
$ echo E > test
$ cat test
E
$ git reset --merge B
error: Entry 'test' not uptodate. Cannot merge.
fatal: Could not reset index file to revision 'B'.
$ cat test
E
```
merge모드와 keep모드의 reset 차이점은 merge conflict를 처리하는 방식이다. 테스트를 위해 다음과 같은 이력을 만들어 보자.
![git keep vs merge](/assets/img/posts/2020-08-18-git-rollback-git-reset-keep-merge.png)
```bash
$ cat test
D
$ git checkout -b dev
$ echo dev > test
$ git commit -a -m "echo dev"
$ git checkout master
$ echo E > test
$ git commit -a -m "echo E" && git tag E
$ git merge dev
Auto-merging test
CONFLICT (content): Merge conflict in test
Automatic merge failed; fix conflicts and then commit the result.
$ cat test
<<<<<<< HEAD
E
=======
dev
>>>>>>> dev
```
test파일을 dev 브랜치와 master 브랜치에서 수정했기에 merge conflict가 발생했다. 이 상태에서 E로 reset을 진행해보자.
```bash
$ git reset --keep E
fatal: Cannot do a keep reset in the middle of a merge.
$ cat test
<<<<<<< HEAD
E
=======
dev
>>>>>>> dev
$ git reset --merge E
$ cat test
E
``````bash
$ echo A > test1
$ git add test1
$ git commit -m "echo A > test1"
$ echo B > test2
$ git add test2
$ git commit -m "echo B > test2"
```
keep모드의 경우 merge conflict를 해결 한 이후 reset을 진행해야 하는 반면, merge모드의 경우 merge conflict를 해결하지 않아도 reset을 진행할 수 있다.

### reset 취소
reset의 취소 역시 reset명령어를 이용한다. git은 reflog라는 명령어를 제공함으로써 HEAD의 변경이력을 조회할 수 있게 해준다. reflog와 reset명령어를 이용해 reset을 취소할 수 있다.
```bash
$ cat test
D
$ echo E > test
$ git reset --hard B
$ cat test
B
$ git reflog
eeddf30 (HEAD -> master, tag: B) HEAD@{0}: reset: moving to B
...
$ git reset --hard HEAD{0}
$ cat test
D
```
하지만 위와 같이 작업 내역을 저장하지 않고 hard모드로 reset했을 경우 E의 내용을 복구할 방법이 없다. 때문에 hard모드로 reset 전 commit, branch 또는 stash를 이용해 현재 작업 내용을 되돌릴 수 있는 안전장치를 만든 후 reset해야 할 것이다.

## revert
다음과 같은 이력을 가지고 있을 때, revert 명령 수행 시 동작을 알아보자.
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
revert명령을 실행해보자.
```bash
$ git revert B
Auto-merging test
CONFLICT (content): Merge conflict in test
error: could not revert eeddf30... echo B
hint: after resolving the conflicts, mark the corrected paths
hint: with 'git add <paths>' or 'git rm <paths>'
hint: and commit the result with 'git commit'
$ cat test
<<<<<<< HEAD
D
=======
A
>>>>>>> parent of e5553e73... echo B
```
revert는 돌아가는 것이 아니라 취소하는 것이다. B를 취소하면서 test 파일에 A가 기록되고 현재 HEAD의 test 파일에는 D가 기록되어 있기 때문에 충돌이 발생한다. 만약 revert를 이용해 B로 돌아고 싶을 경우 D와 C에 해당하는 커밋을 취소하면 된다.
```bash
$ git revert D
Revert "echo D"

This reverts commit d55ebaa.

# Please enter the commit message for your changes. Lines starting
# with '#' will be ignored, and an empty message aborts the commit.
#
# On branch master
# Changes to be committed:
#       modified:   test
$ git revert C
Revert "echo C"

This reverts commit 6eed5cd.

# Please enter the commit message for your changes. Lines starting
# with '#' will be ignored, and an empty message aborts the commit.
#
# On branch master
# Changes to be committed:
#       modified:   test

$ cat test
B
```
다음과 같이 D와 C의 커밋을 취소했다는 커밋이 생기고 test파일에는 B가 기록되어 있을 것이다.
![after git revert](/assets/img/posts/2020-08-18-git-rollback-git-revert-after.png)

취소해야할 커밋이 많다면 git revert HEAD~3와 같이 HEAD를 기준으로 몇 개의 커밋을 취소할지를 지정하거나 git revert -n master~5..master~2와 같이 커밋의 범위를 지정하면 된다.  
revert수 만큼 커밋이 생기는 것이 싫다면 --no-commit 옵션을 사용하자.

## 마무리
그럼 reset과 revert 중 어느 것을 사용해야 할까?  
다음과 같이 origin/master는 C를 가르키고 있는 상태에서 B로 롤백 후 B`를 기록한다고 가정하자.
![after git revert origin](/assets/img/posts/2020-08-18-git-rollback-git-revert-origin.png)

revert를 사용할 경우 HEAD를 과거로 돌리는 것이 아니기에 origin/master와 병합이 쉽다.
![after git revert origin revert](/assets/img/posts/2020-08-18-git-rollback-git-revert-origin-revert.png)

반면 reset을 사용할 경우 다음과 같은 상태가 되며, origin/master와 병합할때 발생하는 충돌을 해결해야 한다. `force 옵션으로 충돌을 해결하지 않고 push하는 짓은 절대로 하지 말자.`
![after git revert origin reset](/assets/img/posts/2020-08-18-git-rollback-git-revert-origin-reset.png)

이 때문에 다른 브랜치와의 병합이 필요한 경우라면 reset보다는 revert를 사용하는 것이 편하다.