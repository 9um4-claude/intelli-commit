---
name: acommit
description: 변경사항을 도메인별로 hunk 단위 분리해 커밋하고 브랜치를 자동 관리. "/acommit", "커밋해줘", "커밋 정리해줘" 등 커밋 요청 시 트리거.
---

# acommit — Smart Commit Assistant

## 실행 순서

1. **컨텍스트 수집** — 저장소 상태, 최근 커밋, 플랫폼, 브랜치 목록
2. **변경 분석 + 플랜 생성** — `acommit:acommit-analyze` 에이전트에 위임
3. **플랜 출력 + 사용자 승인** — 승인 전 git 조작 금지
4. **GPG 확인** — gpg-agent 캐시 확인, 필요 시 unlock 안내
5. **실행 + 리포트 + PR** — `acommit:acommit-exec` 에이전트에 위임

---

## Phase 1: 컨텍스트 수집

```bash
git remote get-url origin 2>/dev/null || echo "NO_REMOTE"
git branch --show-current
git branch -a
git log --oneline -15
git diff HEAD -U3
git diff --cached --name-status
git ls-files --others --exclude-standard
```

플랫폼 감지: `github.com` → `gh` / `gitlab.com` → `glab` / `bitbucket.org` → REST API / `NO_REMOTE` → PR 비활성화

수집한 정보 전체를 `acommit:acommit-analyze` 에이전트에 전달한다.

---

## Phase 2–3: 분석 + 플랜 + 승인

`acommit:acommit-analyze` 에이전트 실행 → 커밋 플랜 수신 → 사용자에게 출력.
`y` 또는 승인 확인 전까지 대기. 수정 요청 시 에이전트 재실행.

---

## Phase 4: GPG 확인

```bash
echo "acommit-gpg-check" | gpg --sign --batch --quiet --output /dev/null 2>&1
echo "EXIT:$?"
```

exit 非0 시 안내 후 대기:
```
별도 터미널에서 실행 후 Enter:
  echo "unlock" | gpg --sign --armor --output /dev/null
```
Enter 후 재확인, 성공할 때까지 반복.

---

## Phase 5–7: 실행 + 리포트 + PR

승인된 플랜 + 플랫폼 정보 + pr 플래그를 `acommit:acommit-exec` 에이전트에 전달.

---

## 핵심 원칙

- `dev`가 개발 기준선. 신규 브랜치는 반드시 `dev`에서 분기. `main`은 build/deploy 전용.
- 실행 전 반드시 `git stash -u`. 실패 시 `git stash pop` 복구 후 보고.
- 도메인 분리 불확실 시 사용자에게 먼저 확인.
