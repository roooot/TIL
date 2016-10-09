# Git Basic

### Git 설정
Git 에 사용할 기본적인 설정이다. `--global` option 을 빼면 해당 repo 에만 적용이 된다.
```sh
git config --global user.name "Root Lee"
git config --global user.email jinhwan.lee@me.com
git config --global core.editor vim
```

### Git 기본 사용법
```sh
# Git repository 를 초기화 한다. '.git' 폴더가 생성된다.
git init

# 현재 local repository 상태 표시
git status

# Staging area 에 파일 추가. (commit 전 임시 영역)
git add <file> ...

# Staging area 에 기록된 파일을 repository 에 내역 반영.
git commit -m 'commit message'

# repository 로그 확인
git log
# 특정 파일과 관련된 로그 확인
git log <file>
# 변경된 내역도 함께 확인
git log -p
# 시각적인 log
git log --graph

# 최신 commit 과 working tree 와의 차이 확인
git diff

# branch 목록을 표시 및 현재 branch 확인
git branch
# local 과 remote branch 모두 표시
git branch -a # --all

# remote 저장소 추가
git remote add origin git@github.com:사용자명/저장소이름.git
```

### Git Alias
아래와 같이 긴 명령어를 간단하게 줄일 수 있다.
```sh
git config --global alias.co checkout
git config --global alias.br branch
git config --global alias.ci commit
git config --global alias.st status
# git reset HEAD 과 동일한 명령어이다.
git config --global alias.unstage 'reset HEAD --'
# 최근 한 개의 commit 을 간단하게 확인 할 수 있다.
git config --global alias.last 'log -1 HEAD'
```

### Fetch vs Pull 차이점
* **fetch** : 리모트로부터 로컬로 가져오지만 merge 하지는 않음
* **pull** : 리모트로부터 가져오고 merge 까지 진행 (fetch + merge)

### 다시 commit 하기
바로 직전의 commit 을 다시 하고 싶을 때, `--amend` 옵션을 붙여 commit 을 한다.
```sh
git commit -m ‘initial commit’
git add forgotten_file
# forgetten_file 을 빼먹고 commit 을 했고 바로 직전의 'initial commit' 에 추가 하고 싶을 때
git commit --amend
```

### Staging Area 에서만 제거하고 working 에서는 남겨두기
Local 에 있는 파일은 남겨두고, git 에서 제거한다.
```sh
git rm --cached <file>
```

### 파일 상태를 Unstage 로 변경하기
stage 했던 파일을 다시 unstage 한다. 위의 `git rm --cached` 와 다른 명령어이다. 둘 다 local에 파일은 유지되지만, `git reset HEAD` 는 staging area 에서만 제거할 뿐 git directory 에는 영향을 주지 않는다. 하지만, `git rm --cached` 는 앞으로의 git 에서 추적을 하지 않도록 제거한다는 명령어로 commit 을 하게 되면 git directory 에서 제거된다.
```sh
git reset HEAD <file>
```

### branch 와 commit log 를 함께
```sh
git log --decorate
```
### commit 내용 검색하기
`function_name` 으로 변경된 내용을 모두 검색한다. (파일 내부 변경까지 모두 검색)
```sh
git log -Sfunction_name
```

### Checkout
checkout 은 원하는 branch 로 변경하는 명령어이다.

새로운 branch 를 생성하면서 해당 branch 로 checkout 할 수도 있다.
```sh
git checkout -b <branch>
```
위 명령어는 아래 두 개의 명령어와 동일하다.
```sh
git branch <branch>   # branch 생성
git checkout <branch> # 해당 branch 로 변경
```

```sh
# remote 으로부터 해당 branch 추적
git checkout --track <remote>/<branch>
# remote branch 와 다른 이름으로 local branch 의 이름을 지정할 수 있다.
git checkout -b <branch> <remote>/<branch>
```

### Push
```sh
git push <remote> <branch>:<remote branch>
git push -u origin featureB:featureBee
# -u(--set-upstream) : 앞으로 해당 branch 를 추적하도록 하여 push/pull 을 편하게 할 수 있다.
```

### Branch 삭제
```sh
git branch -d <branch>
```
Remote 의 branch 삭제는 push 를 통해서 제거가 가능하다
```sh
git push origin --delete <branch>
```
