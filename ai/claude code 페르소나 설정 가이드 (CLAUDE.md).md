# Claude Code 페르소나 설정 가이드 (CLAUDE.md)

## 개요

Claude Code의 행동 방식, 코딩 스타일, 분석 관점 등은 **CLAUDE.md** 파일로 제어한다.
이 파일에 작성한 내용은 Claude가 매 세션마다 읽는 **지시사항(메모리)**으로 작동한다.

---

## 1. CLAUDE.md 파일 위치와 범위

| 종류 | 경로 | 범위 | 공유 |
|:-----|:-----|:-----|:-----|
| 사용자 메모리 | `~/.claude/CLAUDE.md` | 모든 프로젝트 공통 | 본인만 |
| 사용자 규칙 | `~/.claude/rules/*.md` | 모든 프로젝트 공통 | 본인만 |
| 프로젝트 메모리 | `./CLAUDE.md` 또는 `./.claude/CLAUDE.md` | 해당 프로젝트 | 팀 (git 커밋) |
| 프로젝트 로컬 | `./CLAUDE.local.md` | 해당 프로젝트 | 본인만 (자동 gitignore) |
| 프로젝트 규칙 | `./.claude/rules/*.md` | 해당 프로젝트 | 팀 (git 커밋) |
| 조직 정책 | `/Library/Application Support/ClaudeCode/CLAUDE.md` (macOS) | 전체 조직 | 관리자 배포 |

### 우선순위 (위가 높음)

1. 조직 정책 (관리자 배포, 사용자가 덮어쓸 수 없음)
2. 프로젝트 로컬 (`CLAUDE.local.md`)
3. 프로젝트 메모리 (`./CLAUDE.md`)
4. 사용자 메모리 (`~/.claude/CLAUDE.md`)

### 디렉토리 탐색 동작

- **상위 디렉토리**: 작업 디렉토리에서 루트까지 올라가며 모든 CLAUDE.md를 자동 로드
- **하위 디렉토리**: Claude가 해당 디렉토리의 파일을 읽을 때 on-demand로 로드
- 모노레포에서 유용: `root/CLAUDE.md`와 `root/app/CLAUDE.md` 모두 반영됨

---

## 2. 작성 형식

일반 Markdown으로 작성한다. 특별한 형식이 필요 없다.

```markdown
# 코드 스타일
- ES modules (import/export) 사용, CommonJS (require) 금지
- 함수형 컴포넌트 우선

# 빌드 & 테스트
- Build: npm run build
- Test: npm run test -- --testPathPattern=<file>
- Lint: npm run lint

# Git 규칙
- 커밋 메시지는 한국어로 작성
- 브랜치명: feature/기능명, fix/버그명
```

### 특수 문법

**`@import` - 다른 파일 참조:**

```markdown
프로젝트 개요는 @README.md 참고.
Git 워크플로우: @docs/git-workflow.md
개인 설정: @~/.claude/my-overrides.md
```

- 상대 경로, 절대 경로 모두 가능
- 최대 5단계까지 재귀 import 지원
- 코드 블록 안의 `@참조`는 무시됨

**강조 표현 - 준수율 향상:**

```markdown
IMPORTANT: 테스트 없이 커밋하지 말 것
YOU MUST: 모든 API 응답에 에러 핸들링을 포함할 것
```

---

## 3. 페르소나 지정 방법

### 방법 1: 사용자 CLAUDE.md에 기본 페르소나 설정

`~/.claude/CLAUDE.md`에 작성하면 모든 프로젝트에 적용된다.

```markdown
# 기본 페르소나
- 한국어로 응답
- 설명은 간결하게, 코드 위주로
- 기술적 판단의 근거를 항상 제시
- 트레이드오프를 명시

# 코드 리뷰 관점
- 보안 취약점 우선 확인
- 성능 영향도 언급
- 대안이 있으면 함께 제시
```

### 방법 2: 프로젝트 CLAUDE.md에 프로젝트별 페르소나 설정

`./CLAUDE.md`에 작성하면 해당 프로젝트에서만 적용된다.

```markdown
# 페르소나
너는 시니어 Android 아키텍트야.

# 코드 리뷰 기준
- Clean Architecture 레이어 침범 여부 확인
- ViewModel 비대화 경고
- DI scope 문제 감지
- Coroutine 오용 패턴 지적
- Memory leak 가능성 분석

# 프로젝트 컨벤션
- MVVM + Clean Architecture
- Hilt 사용
- Coroutine + Flow 기반 비동기 처리
```

### 방법 3: CLAUDE.local.md로 개인 오버라이드

팀 프로젝트에서 개인 취향만 따로 설정할 때 사용한다.
`CLAUDE.local.md`는 자동으로 `.gitignore`에 추가된다.

```markdown
# 개인 설정
- 디버그 시 Logcat 출력 포함해서 설명
- 테스트용 API endpoint: http://localhost:8080
- 커밋 메시지 스타일: conventional commits
```

### 방법 4: rules 디렉토리로 모듈 분리

규칙이 많아지면 `.claude/rules/`에 주제별로 분리한다.

```
.claude/rules/
├── code-style.md       # 코드 스타일 규칙
├── testing.md          # 테스트 작성 규칙
├── security.md         # 보안 관련 규칙
└── api-design.md       # API 설계 원칙
```

규칙 파일에 `paths` frontmatter를 추가하면 특정 파일에만 적용할 수 있다:

```yaml
---
paths:
  - "src/api/**/*.ts"
  - "lib/**/*.ts"
---

# API 규칙
- 모든 엔드포인트에 입력 검증 포함
- 표준 에러 응답 포맷 사용
```

---

## 4. 작성 원칙

### 넣어야 할 것

- Claude가 추측할 수 없는 빌드/테스트/린트 명령어
- 언어 기본값과 다른 코드 스타일 규칙
- 프로젝트 고유 아키텍처 결정사항
- 브랜치 네이밍, PR 컨벤션 등 팀 규칙
- 개발 환경 특이사항 (필수 환경변수 등)

### 넣지 말아야 할 것

- Claude가 코드를 읽으면 알 수 있는 정보
- 언어의 표준 컨벤션 (Claude가 이미 알고 있음)
- 자주 변경되는 정보
- 파일별 코드베이스 설명
- "깨끗한 코드를 작성해라" 같은 자명한 지시

---

## 5. 실전 구성 예시

```
프로젝트 루트/
├── CLAUDE.md                  # 팀 공유 (git 커밋)
├── CLAUDE.local.md            # 개인 설정 (gitignore)
└── .claude/
    ├── CLAUDE.md              # ./CLAUDE.md 대체 위치
    └── rules/
        ├── android.md         # Android 개발 규칙
        ├── testing.md         # 테스트 규칙
        └── security.md        # 보안 규칙

홈 디렉토리/
└── .claude/
    ├── CLAUDE.md              # 전체 프로젝트 공통 설정
    └── rules/
        └── personal-style.md  # 개인 코딩 스타일
```

---

## 6. 유용한 명령어

| 명령어 | 설명 |
|:-------|:-----|
| `/init` | 프로젝트 구조를 분석해 CLAUDE.md 초안 자동 생성 |
| `/memory` | 현재 로드된 메모리 파일 목록 확인 |

---

## 7. 요약

| 항목 | 내용 |
|:-----|:-----|
| 파일 형식 | Markdown (특별한 문법 불필요) |
| 개인 전역 설정 | `~/.claude/CLAUDE.md` |
| 프로젝트 팀 설정 | `./CLAUDE.md` 또는 `.claude/CLAUDE.md` |
| 개인 로컬 설정 | `./CLAUDE.local.md` |
| 모듈 분리 | `.claude/rules/*.md` |
| 파일 참조 | `@파일경로` 구문 |
| 핵심 원칙 | Claude가 스스로 알 수 없는 것만 명시하고, 간결하게 유지 |
