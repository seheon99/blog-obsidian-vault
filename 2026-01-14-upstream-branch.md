---
title: Git의 Upstream Branch
type: write-up
topics:
  - Git
description: 새 브랜치를 만들 때면 항상 업스트림 설정을 한다. 업스트림은 왜 설정할까?
createdAt: 2026-01-14T17:45:38+09:00
---
```bash
git checkout -b feature/login
git push -u origin feature/login
```

# Upstream

- Git은 브랜치를 흐름(stream)에 비유한다
  - 새로운 브랜치를 만들면 기존 흐름에 이어지는 새로운 흐름을 만들고
  - 기존 흐름을 **upstream** 이라고 부른다
- 이 설정은 `branch.<name>.remote` 와 `branch.<name>.merge` 로 `.git/config` 에 저장된다
- 이 설정을 통해 Git은 `git status` 과 `git branch -vv` 에서 현재 브랜치의 기준점을 잡고 ahead, behind 정보를 계산한다
  - **ahead** 는 새로운 브랜치에만 있는 커밋
  - **behind** 는 기존 브랜치에만 있는 커밋
  - 평소에 ahead, behind 수가 생각과 다르게 나왔다면 업스트림 설정을 안한 것!!

```
Your branch is ahead of 'origin/dev' by 2 commits.
Your branch is behind 'origin/dev' by 1 commit.
```

- 또한 매개변수 없이 사용하는 `git pull` 과 `git push` 명령에서도 사용된다
  - `git pull` $\approx$ `git fetch <branch.dev.remote> && git merge <branch.dev.merge>`
  - 매개변수 없이 사용하는 `git push` 명령도 업스트림의 정보를 통해 타겟을 결정한다

## 명령어 살펴보기

### `git branch <branchname>`

- `<branchname>` 의 이름으로 새로운 브랜치를 만든다

### `git branch [--set-upstream | --track | --no-track] <branchname>`

- `<branchname>` 의 이름으로 새로운 브랜치를 만든다
- 새로 만든 브랜치의 upstream 을 현재 브랜치로 설정한다
  **.git/config** (main에서 dev를 만들때)

```
[branch "dev"]
	remote = .
	merge = refs/heads/main
```

- `branch.<branchname>.remote` 가 로컬(`.`)로 설정된다
  - `git status` 에서 ahead와 behind가 원격 레포가 아닌 로컬을 기준으로 계산된다
  - `git pull` 에서 원격 레포가 아닌 로컬에서 가져온다

> [!WARNING]
> 이 상태를 모르고 `git pull` 을 하면 원격 레포의 dev 브랜치 코드를 **가져오는게** 아닌 로컬의 main 브랜치에서 코드를 **병합하게** 된다.

### `git branch [--set-upstream-to=<upstream> | -u <upstream>) [<branchname>]`

- 브랜치의 업스트림을 다시 설정한다
- `-u <upstream>` 을 통해 업스트림을 원격 레포로 바꿀 수 있다
  **.git/config** (`git branch -u origin/dev dev`)

```
[branch "dev"]
	remote = origin
	merge = refs/heads/dev
```

### `git push -u <remote> <branch>`

- 현재 브랜치의 내용을 `remote` 의 `branch` 로 업로드한다
- 또한 업스트림 설정을 다음과 같이 한다

```
[branch <current_branch>]
	remote = <remote>
	merge = refs/heads/<branch>
```

## 그래서

업스트림은 Git이 어떤 흐름을 기준으로 비교하고 어떻게 동작할지 결정하는 설정이다.
업스트림 설정이 잘못되면

- 아무리 `push`, `pull` 을 해도 `git status` 에서 ahead와 behind가 바뀌지 않고
- `git pull` 은 생각과 완전히 다르게 동작할 수도 있다
  그래서 새 브랜치를 만들게 되면 `git push -u` 를 통해 업스트림을 올바르게 잡아줘야 한다.
