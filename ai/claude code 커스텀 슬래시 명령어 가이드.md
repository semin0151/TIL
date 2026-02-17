# Claude Code 커스텀 슬래시 명령어 가이드

Claude Code에서는 `/명령어` 형태의 슬래시 명령어를 직접 만들 수 있다.
이를 **Skills**라고 부르며, 반복되는 작업 지시를 재사용 가능한 프롬프트 템플릿으로 만드는 기능이다.

---

## 1. 저장 위치

| 범위 | 경로 | 적용 대상 |
|:-----|:-----|:----------|
| 개인(글로벌) | `~/.claude/skills/<명령어명>/SKILL.md` | 모든 프로젝트 |
| 프로젝트 | `.claude/skills/<명령어명>/SKILL.md` | 해당 프로젝트만 |

> 레거시 경로인 `~/.claude/commands/<명령어명>.md`, `.claude/commands/<명령어명>.md`도 여전히 동작한다.

---

## 2. 파일 구조

각 명령어는 **디렉토리 + SKILL.md** 파일로 구성한다.

```
~/.claude/skills/
└── review/
    └── SKILL.md        # 필수 - 명령어의 본체
```

`SKILL.md`는 두 부분으로 나뉜다:

1. **YAML frontmatter** - 명령어 메타데이터 (선택)
2. **Markdown 본문** - Claude에게 전달할 지시 내용

```yaml
---
name: review
description: 코드 변경사항을 리뷰한다
---

아래 코드 변경사항을 리뷰해줘: $ARGUMENTS

검토 항목:
1. 버그 및 로직 오류
2. 보안 취약점
3. 성능 이슈
```

이렇게 만들면 `/review` 명령어가 즉시 사용 가능해진다.

---

## 3. Frontmatter 옵션

모든 필드는 선택사항이다.

| 필드 | 설명 |
|:-----|:-----|
| `name` | 슬래시 명령어 이름. 디렉토리명이 기본값 |
| `description` | 명령어 설명. Claude가 자동 호출 여부를 판단하는 데 사용 |
| `argument-hint` | 자동완성 시 표시될 인자 힌트 (예: `[파일명]`) |
| `disable-model-invocation` | `true`로 설정하면 Claude의 자동 호출을 차단. 수동 `/명령어`로만 실행 |
| `user-invocable` | `false`로 설정하면 `/` 메뉴에서 숨김. Claude만 자동으로 호출 가능 |
| `allowed-tools` | 이 명령어 실행 시 허용할 도구 (예: `Read, Grep, Bash(gh *)`) |
| `context` | `fork`으로 설정하면 별도 서브에이전트 컨텍스트에서 실행 |

### 호출 방식 제어

| 설정 | 사용자 호출 | Claude 자동 호출 |
|:-----|:-----------|:----------------|
| 기본값 (설정 없음) | O | O |
| `disable-model-invocation: true` | O | X |
| `user-invocable: false` | X | O |

---

## 4. 변수와 템플릿

### 인자 변수

| 변수 | 설명 |
|:-----|:-----|
| `$ARGUMENTS` | 명령어 뒤에 입력한 모든 인자 |
| `$0`, `$1`, `$2` ... | 위치별 개별 인자 (0부터 시작) |
| `${CLAUDE_SESSION_ID}` | 현재 세션 ID |

**예시 - 단일 인자:**

```yaml
---
name: fix-issue
description: GitHub 이슈를 수정한다
disable-model-invocation: true
---

GitHub 이슈 #$ARGUMENTS 를 수정해줘.
1. 이슈 내용을 읽고
2. 수정 사항을 구현하고
3. 테스트를 작성해줘
```

`/fix-issue 42` 실행 시 `$ARGUMENTS`가 `42`로 치환된다.

**예시 - 복수 인자:**

```yaml
---
name: migrate
description: 컴포넌트를 다른 프레임워크로 마이그레이션한다
---

$0 컴포넌트를 $1에서 $2로 마이그레이션해줘.
기존 동작과 테스트를 모두 유지해야 해.
```

`/migrate SearchBar React Vue` 실행 시 각 위치 변수에 대입된다.

### 셸 명령어 주입

`` !`명령어` `` 구문으로 셸 명령어 실행 결과를 프롬프트에 삽입할 수 있다.
명령어는 Claude에게 전달되기 **전에** 실행되어 결과값으로 치환된다.

```yaml
---
name: pr-review
description: 현재 PR을 리뷰한다
context: fork
allowed-tools: Bash(gh *)
---

## PR 컨텍스트
- 변경된 파일: !`gh pr diff --name-only`
- PR diff: !`gh pr diff`

위 변경사항을 리뷰해줘.
```

---

## 5. 실전 예시

### 개인용: Android 아키텍처 리뷰

```
~/.claude/skills/android-review/SKILL.md
```

```yaml
---
name: android-review
description: Android 프로젝트의 아키텍처를 리뷰한다
disable-model-invocation: true
allowed-tools: Read, Grep, Glob
---

이 프로젝트의 아키텍처를 리뷰해줘.

검토 항목:
1. Clean Architecture 레이어 침범 여부
2. ViewModel 비대화
3. DI scope 문제
4. Coroutine 오용 (GlobalScope, Dispatcher 오용, blocking call)
5. Memory leak 가능성
```

### 프로젝트용: 커밋 전 체크리스트

```
.claude/skills/pre-commit-check/SKILL.md
```

```yaml
---
name: pre-commit-check
description: 커밋 전 코드 품질을 점검한다
disable-model-invocation: true
allowed-tools: Read, Grep, Glob, Bash(git diff *)
---

현재 스테이징된 변경사항을 점검해줘.

## 변경 내용
!`git diff --cached`

## 점검 항목
1. 하드코딩된 시크릿이나 API 키가 없는지
2. console.log / print 등 디버그 코드가 남아있지 않은지
3. TODO/FIXME 주석이 새로 추가되지 않았는지
4. 테스트가 필요한 변경인지
```

### 백그라운드 지식 (자동 로드용)

```
.claude/skills/project-context/SKILL.md
```

```yaml
---
name: project-context
description: 프로젝트의 레거시 시스템에 대한 컨텍스트. 레거시 코드 관련 작업 시 참고.
user-invocable: false
---

이 프로젝트의 레거시 결제 시스템은 XML 기반이며...
(프로젝트 고유의 맥락 정보를 여기에 기술)
```

---

## 6. 요약

| 항목 | 내용 |
|:-----|:-----|
| 저장 위치 | `~/.claude/skills/<name>/SKILL.md` (개인) 또는 `.claude/skills/<name>/SKILL.md` (프로젝트) |
| 파일 형식 | YAML frontmatter + Markdown |
| 호출 방법 | `/<name>` 또는 `/<name> 인자` |
| 인자 전달 | `$ARGUMENTS`, `$0`, `$1` 등 |
| 동적 데이터 | `` !`셸 명령어` `` 구문 |
