---
name: intellicommit
description: 변경사항을 도메인별로 hunk 단위 분리해 커밋하고 브랜치를 자동 관리. "/intellicommit", "커밋해줘", "커밋 정리해줘" 등 커밋 요청 시 트리거.
---

# IntelliCommit — Smart Commit Assistant

## 실행 순서

1. **컨텍스트 수집** — 저장소 상태, 브랜치, 플랫폼
2. **변경 개요 파악** — `--stat`으로 파일 목록 확인
3. **도메인 분류 + 관련 파일만 full diff 요청**
4. **플랜 생성 + 사용자 승인**
5. **patch 파일 생성** (stash 전 필수)
6. **GPG 확인**
7. **실행 + 리포트 + PR** — `intelli-commit:intellicommit-exec` 에이전트에 위임

---

## Phase 1: 컨텍스트 수집

```bash
git remote get-url origin 2>/dev/null || echo "NO_REMOTE"
git branch --show-current
git branch --sort=-committerdate | head -15
git log --oneline -7
git diff HEAD --stat
git diff HEAD --name-only
git ls-files --others --exclude-standard
```

플랫폼 감지: `github.com` → `gh` / `gitlab.com` → `glab` / `bitbucket.org` → REST API / `NO_REMOTE` → PR 비활성화

---

## Phase 2: 도메인 분류

최근 커밋 히스토리로 진행 중인 작업 흐름 파악. `--name-only` 목록을 도메인별로 분류한다.

**분리 기준:**
- 서로 다른 기능/시스템 → 별도 커밋
- 기능 구현 + 리팩토링 혼재 → 분리
- config/meta + 기능 변경 → 분리 고려
- 불확실 시 하나로 묶고 사용자에게 확인

도메인이 확정되면 해당 파일들에 대해서만 full diff 요청:

```bash
git diff HEAD -U3 -- <file1> <file2> ...
```

untracked 파일은 Phase 1의 `git ls-files --others` 결과 사용.

---

## Phase 3: 플랜 생성 + 승인

### 브랜치 배정
1. 기존 브랜치에서 같은 도메인 → 재사용
2. 없음 → 신규 (`dev` 기준, `dev` 없으면 `main`)
3. 현재 브랜치가 해당 도메인 → 현재 브랜치 사용

브랜치명: kebab-case 소문자 30자 이내, type prefix 없음 (`rhythm-judgment`, `inventory-ui`)

### 커밋 메시지 형식
```
<type>: <요약>

<왜 이 변경이 필요했는지>

- <세부 내용>
```

아래는 절대로 메시지에 포함하지 않는다.
```
Co-Authored-By: ...
Co-written by ...
Generated with Claude ...
🤖 ...
```

type: `feat` `fix` `refactor` `perf` `config` `chore` `test` `docs` — "왜/영향" 중심으로 작성.

### 플랜 출력 형식
```
[흐름] <최근 커밋 기반 1문장>

커밋 1/N → <branch> (신규|기존)
<type>: <요약>
파일: <file1>, <file2> (hunk N)
```

`y` 또는 승인 전까지 대기. 수정 요청 시 재분류.

승인 직후 **fast-path 조건을 확인**한다:

> 플랜의 모든 커밋 그룹이 현재 브랜치를 대상으로 하는가?

**YES** → Phase 4~6 생략, 아래 fast-path 실행:

그룹 내 분리 단위에 따라 스테이징 방식이 다르다:

- **파일 단위 분리** (그룹의 대상 파일이 다른 경우):
  ```bash
  git add <file1> <file2>
  git commit -m "<커밋 메시지>"
  ```

- **hunk 단위 분리** (같은 파일 안에서 나눠야 하는 경우):
  patch 파일을 생성한 뒤 인덱스에 직접 적용한다 (stash/브랜치 전환 없음):
  ```bash
  git apply --cached group_N.patch
  git commit -m "<커밋 메시지>"
  ```

그룹 수만큼 반복 후 아래 리포트를 출력하고 종료 (exec 에이전트 호출 없음):
```
[흐름] <최근 커밋 포함 진행 상황 1~2문장>
✓  <현재 브랜치>  →  <커밋 제목>
```

**NO** → Phase 4로 진행 (stash+patch 경로).

---

## Phase 4: patch 파일 생성 + 사전 검증 (stash 전 필수)

승인된 플랜 기준으로 각 커밋 그룹의 patch 파일을 **stash 전에** 생성한다.
- 해당 그룹의 hunk만 추출 (`diff --git`, `---`, `+++` 헤더 포함)
- `group_N.patch`로 저장
- untracked 파일은 내용을 변수로 보관

patch를 생성한 즉시 (다음 그룹으로 넘어가기 전) 두 단계로 검증한다.

**단계 1 — 형식 검증:**
```bash
git apply --check --verbose group_N.patch 2>&1
```
오류 메시지에 `corrupt patch` / `bad header` / `invalid line` 등이 포함되면
**형식 오류**로 판단. patch를 재생성한다 (최대 1회 재시도).

**단계 2 — 적용 가능성 검증:**
형식이 올바른데도 exit 非0이면 **컨텍스트 불일치**. 대상 브랜치가 기존 브랜치이고
dev로부터 많이 갈라진 경우 발생 가능.
→ 사용자에게 원인과 충돌 hunk를 보고하고 **stash 없이 중단**.
→ Phase 3으로 돌아가 해당 그룹의 브랜치 배정을 재검토한다.

---

## Phase 5: 서명 확인

먼저 서명 방식을 감지한다:
```bash
git config gpg.format 2>/dev/null || echo "gpg"
```

**SSH 서명** (`ssh` 출력 시):
```bash
ssh-add -l 2>&1
```
exit 非0 또는 "no identities" 출력 시 안내 후 대기:
```
별도 터미널에서 실행 후 Enter:
  ssh-add ~/.ssh/id_ed25519   # 또는 user.signingkey 경로
```
Enter 후 재확인, 성공할 때까지 반복.

**GPG 서명** (`gpg` 또는 미설정 시):
```bash
echo "intelliCommit-gpg-check" | gpg --sign --batch --quiet --output /dev/null 2>&1
echo "EXIT:$?"
```
exit 非0 시 안내 후 대기:
```
별도 터미널에서 실행 후 Enter:
  echo "unlock" | gpg --sign --armor --output /dev/null
```
Enter 후 재확인, 성공할 때까지 반복.

---

## Phase 6–8: 실행 + 리포트 + PR

생성된 **patch 파일 경로 목록** + 플랜 + 플랫폼 정보 + pr 플래그를 `intelli-commit:intellicommit-exec` 에이전트에 전달.

---

## 핵심 원칙

- `dev`가 개발 기준선. 신규 브랜치는 반드시 `dev`에서 분기. `main`은 build/deploy 전용.
- patch 파일 생성은 반드시 stash 전.
- 도메인 분리 불확실 시 사용자에게 먼저 확인.
