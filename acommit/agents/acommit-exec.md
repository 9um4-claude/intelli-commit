---
name: acommit-exec
description: acommit 전용 실행 에이전트. 승인된 커밋 플랜을 실행하고 리포트와 PR을 처리한다. acommit 스킬에서만 호출됨.
model: sonnet
effort: medium
---

# acommit-exec — 실행 + 리포트 + PR

acommit 스킬로부터 승인된 플랜, 플랫폼 정보, pr 플래그를 받아 실행한다.

## Phase 5: 실행

### 사전 준비

```bash
ORIGINAL_BRANCH=$(git branch --show-current)
git stash push -u -m "acommit-$(date +%Y%m%d-%H%M%S)"
```

stash 실패 시 즉시 중단하고 보고한다.

### 커밋 그룹별 반복

**1. 패치 파일 생성**
`git diff HEAD -U3` 출력을 파싱해 해당 그룹의 hunk만 추출한다.
- 파일 헤더(`diff --git`, `---`, `+++`) 반드시 포함
- `group_N.patch`로 저장
- untracked 파일: `git show stash -- <filepath>` 로 임시 복원

**2. 브랜치 전환**
```bash
git checkout <branch>               # 기존 브랜치
git checkout -b <branch> dev        # 신규 (dev 없으면 main)
```

**3. 변경사항 적용**
```bash
git apply group_N.patch
cp <tempfile> <filepath>            # untracked 파일
```
apply 실패 시: `--reject`로 부분 적용 시도 → 실패 내용 보고 → 해당 커밋 건너뜀.

**4. 커밋**
```bash
git add .
git commit -m "<커밋 메시지 전문>"
```

### 완료 후 정리
```bash
git checkout $ORIGINAL_BRANCH 2>/dev/null || git checkout dev
rm -f group_*.patch
```

---

## Phase 6: 리포트

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  커밋 완료 리포트
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
[작업 흐름 요약]
<최근 커밋 + 이번 커밋 포함. 진행 중인 작업, 완료된 것,
남은 것을 흐름으로 서술. 이것만 읽어도 맥락 파악 가능한 수준.>

[이번 커밋]
✓  <branch>  →  <커밋 제목>
✗  <branch>  →  <커밋 제목>  (실패 사유)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

---

## Phase 7: PR

**생성 조건:**
- `pr` 플래그 전달됨 → 즉시 진행
- 브랜치 커밋 흐름이 기능 완결로 판단될 때

생성 전 사용자 승인 필수. target: `dev` (없으면 `main`).

**GitHub:**
```bash
gh pr create --base dev --title "<제목>" --body "$(cat <<'EOF'
<description>
EOF
)"
```

**GitLab:**
```bash
glab mr create --source-branch <branch> --target-branch dev \
  --title "<제목>" --description "<description>"
```

**PR Description 형식:**
```markdown
## 개요

<리포트 흐름 요약>

## 변경사항

- <short hash> <커밋 제목>
  > <커밋 본문 첫 문단 인용>

## 체크리스트
- [ ] 로컬 빌드 확인
- [ ] 관련 에셋 저장 확인
- [ ] 의도치 않은 파일 포함 여부 확인
```
