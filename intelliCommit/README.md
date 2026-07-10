# IntelliCommit

변경사항을 도메인별로 hunk 단위로 분리해 각 브랜치에 커밋하고 PR까지 자동으로 처리하는 Claude Code 플러그인.

## 핵심 기능

**도메인별 자동 분류**
`git diff`의 hunk를 분석해 서로 다른 기능/시스템 단위로 묶는다. 같은 파일 안에 두 도메인이 섞여 있어도 hunk 단위로 분리한다.

**다중 브랜치 커밋 라우팅**
도메인마다 적절한 브랜치를 배정하고 stash+patch 방식으로 각 브랜치에 독립적으로 커밋한다. 기존 브랜치 재사용 또는 `dev` 기준 신규 브랜치 생성을 자동으로 처리한다.

**커밋 메시지 품질 강제**
`feat` / `fix` / `refactor` 등 타입 prefix와 함께 "왜" 이 변경이 필요했는지를 중심으로 메시지를 작성한다.

**PR 자동 생성**
GitHub(`gh`) / GitLab(`glab`) 플랫폼을 자동 감지해 PR/MR을 생성한다.

## 사용법

Claude Code에서 다음과 같이 호출한다:

```
/intellicommit
커밋해줘
커밋 정리해줘
```

`pr` 플래그를 붙이면 커밋 후 PR 생성까지 진행한다:

```
/intellicommit pr
```

## 실행 흐름

```
Phase 1  저장소 상태, 브랜치, 플랫폼 수집
Phase 2  --stat + --name-only로 변경 파일 파악 → 도메인별 분류
Phase 3  플랜 생성 → 사용자 승인
         ↓ 단일 브랜치인 경우 fast-path (Phase 4~6 생략)
Phase 4  patch 파일 생성 + 형식/적용 가능성 검증
Phase 5  GPG/SSH 서명 확인
Phase 6  stash → 브랜치별 patch apply → 커밋 → 복구
Phase 7  커밋 리포트
Phase 8  PR 생성 (플래그 또는 판단 시)
```

## 안전 장치

- **사전 검증**: stash 전에 `git apply --check`로 모든 patch의 적용 가능성을 확인한다. 실패하면 실행하지 않는다.
- **실패 시 전체 롤백**: apply 중 오류 발생 시 건너뛰지 않고 즉시 중단 후 `git stash pop`으로 원상복구한다.
- **Resume**: `.intellicommit-state` 파일로 그룹별 진행 상태를 추적한다. 중단 후 재실행 시 완료된 그룹을 건너뛰고 남은 것만 재개할 수 있다.
- **승인 게이트**: 플랜 확인 없이는 git 조작을 하지 않는다.

## 브랜치 규칙

| 브랜치 | 역할 |
|--------|------|
| `main` | 빌드/배포 전용. 직접 커밋하지 않는다. |
| `dev`  | 개발 기준선. 신규 브랜치는 여기서 분기한다. |
| `<domain>` | 도메인별 작업 브랜치. kebab-case 소문자 30자 이내. |

## 요구사항

- Claude Code
- Git 2.x 이상
- PR 생성 시: `gh` (GitHub) 또는 `glab` (GitLab)

## 라이선스

MIT
