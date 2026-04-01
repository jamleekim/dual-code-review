---
name: dual-code-review
description: "Two-AI code review system where Opus and Codex independently review code, debate differences, and produce a unified report. Invoke with /dual-review. MUST use this skill when the user mentions: dual review, cross review, team review, 교차 검증, 듀얼 리뷰, 팀 리뷰, 코드 리뷰 팀, reviewing from two perspectives, comparing two AI reviews, independent code review by multiple models, opus and codex review, or wants .reviews/ directory reports. Also use when the user asks for multi-perspective code analysis, wants two reviewers to check their code, or mentions cross-checking code changes. Do NOT use for simple single-reviewer code review requests or bug fixes."
---

# Dual Code Review

Opus와 Codex가 각각 독립적으로 코드를 리뷰하고, 서로의 리뷰를 비교/토론하여 최종 결론을 도출하는 코드 리뷰 시스템.

두 모델은 서로 다른 학습 배경과 강점을 가지고 있다. Opus는 아키텍처와 설계 패턴에 강하고, Codex(GPT 기반)는 실용적 버그 탐지와 코드 동작 분석에 강점이 있다. 하나의 모델이 놓칠 수 있는 문제를 다른 모델이 잡아내는 "교차 검증"으로 리뷰 품질을 높인다.

## 실행 모드

이 스킬은 실행 환경에 따라 두 가지 모드로 동작한다.

### Mode A: 메인 세션 (Full Dual Review)

메인 대화 세션에서 실행될 때의 정상 모드. Agent 도구를 사용하여 Opus와 Codex를 **1단계 서브에이전트**로 병렬 생성한다.

```
메인 세션 (오케스트레이터, Agent 도구 ✅)
    ├─ Opus 서브에이전트 (독립 리뷰)
    └─ Codex 서브에이전트 (독립 리뷰)
```

Claude Code의 아키텍처는 **단일 레벨 위임(single-level delegation)**을 따른다. 메인 세션만이 서브에이전트를 생성할 수 있고, 서브에이전트는 또 다른 서브에이전트를 생성할 수 없다. 따라서 이 스킬이 서브에이전트 내에서 호출되면 Mode B로 자동 전환된다.

### Mode B: 서브에이전트 환경 (Sequential Dual Perspective)

서브에이전트 내에서 실행될 때의 폴백 모드. Agent 도구가 없으므로 **두 번의 독립적인 리뷰 패스**를 순차 수행한다.

```
서브에이전트 (Agent 도구 ❌)
    ├─ Pass 1: 아키텍처/설계 관점 리뷰 (Opus 시점)
    └─ Pass 2: 버그/런타임/보안 관점 리뷰 (Codex 시점)
```

Mode B에서는:
- 두 리뷰 패스를 **의식적으로 분리**한다. Pass 1 작성 시 Pass 2의 관점은 배제하고, Pass 2 작성 시 Pass 1과 다른 시각을 의식적으로 추구한다.
- 리포트 헤더에 `**Mode**: Sequential Dual Perspective (서브에이전트 환경)` 을 명시하여 사용자가 실행 모드를 알 수 있게 한다.
- 리뷰 품질은 Mode A보다 낮을 수 있다 (동일 모델이 두 관점을 연기하므로 진정한 독립성이 제한됨).

### 모드 감지 방법

Agent 도구의 사용 가능 여부로 판단한다:
- Agent 도구가 사용 가능 → **Mode A**
- Agent 도구가 사용 불가능 → **Mode B**

## 실행 흐름

```
사용자가 /dual-review 실행
        │
        ▼
┌─ 1. 변경 사항 감지 ────────────────┐
│  git diff로 변경된 파일/코드 수집   │
│  변경 파일의 전체 내용도 읽기       │
│  리뷰 대상 범위 결정                │
└────────────────────────────────────┘
        │
        ▼
┌─ 2. 독립 리뷰 (병렬 실행) ─────────┐
│  ┌─ Opus ──┐   ┌─ Codex ──┐       │
│  │ Agent   │   │ codex:   │       │
│  │ model:  │   │ codex-   │       │
│  │ opus    │   │ rescue   │       │
│  └─────────┘   └──────────┘       │
│  ※ 반드시 같은 턴에서 병렬 실행    │
└────────────────────────────────────┘
        │
        ▼
┌─ 3. 비교 & 토론 ──────────────────┐
│  두 리뷰 결과를 비교               │
│  합의점 / 차이점 도출              │
│  차이점에 대한 토론 및 최종 판단   │
└────────────────────────────────────┘
        │
        ▼
┌─ 4. 리포트 생성 ──────────────────┐
│  .reviews/ 디렉토리에              │
│  PR/MR 스타일 마크다운 리포트 생성 │
└────────────────────────────────────┘
        │
        ▼
┌─ 5. 사용자 승인 ──────────────────┐
│  변경 제안 목록 제시               │
│  사용자 승인 후에만 코드 수정      │
└────────────────────────────────────┘
```

## Step 1: 변경 사항 감지

리뷰 대상을 파악한다. 다음 순서로 범위를 결정:

1. 사용자가 특정 파일이나 범위를 지정한 경우 → 해당 범위 사용
2. `--base <ref>` 옵션이 있는 경우 → `git diff <ref>`
3. `--commits N` 옵션이 있는 경우 → `git diff HEAD~N`
4. 스테이징된 변경이 있는 경우 → `git diff --cached`
5. 작업 트리에 변경이 있는 경우 → `git diff`
6. 위 모두 없는 경우 → `git diff HEAD~1`

### 컨텍스트 수집

diff만으로는 충분한 리뷰가 어렵다. 반드시 다음을 모두 수집:

```bash
# 1. 변경 파일 목록과 통계
git diff --stat [범위]

# 2. 전체 diff
git diff [범위]
```

```
# 3. 변경된 각 파일의 전체 내용을 Read 도구로 읽기
#    diff의 컨텍스트만으로는 주변 코드를 이해할 수 없으므로,
#    각 변경 파일의 전체 내용을 읽어서 리뷰어에게 함께 전달한다.
```

수집한 정보를 하나의 컨텍스트 블록으로 구성하여 두 리뷰어에게 동일하게 전달한다.

## Step 2: 독립 리뷰

### Mode A: 병렬 서브에이전트 실행 (메인 세션)

**반드시 두 에이전트를 동일한 메시지(같은 턴)에서 병렬로 실행한다.** 순차 실행하면 먼저 실행된 리뷰가 나중 리뷰에 영향을 줄 수 있으므로, 독립성을 보장하기 위해 병렬 실행이 핵심이다.

#### Opus 리뷰어

```
Agent(
  description: "Opus code review",
  subagent_type: "general-purpose",
  model: "opus",
  mode: "auto",
  prompt: <리뷰 프롬프트 — diff, 파일 내용, 리뷰 기준 포함>
)
```

#### Codex 리뷰어

Codex는 `codex:codex-rescue` 서브에이전트를 통해 호출한다. Codex 에이전트는 Bash 도구만 사용 가능하므로, 필요한 컨텍스트(diff, 파일 내용)를 프롬프트에 **직접 포함**해야 한다. Codex가 스스로 파일을 읽을 수 없다고 가정하고, 모든 코드를 프롬프트 안에 넣어준다.

```
Agent(
  description: "Codex code review",
  subagent_type: "codex:codex-rescue",
  prompt: <리뷰 프롬프트 — diff, 파일 내용, 리뷰 기준 포함.
           추가 지시: "리뷰 결과를 .reviews/ 디렉토리에 저장하지 말고
           표준 출력으로 반환하라">
)
```

#### Codex 사용 불가 시 대체

Codex 플러그인이 설치되지 않았거나 응답이 없는 경우, Codex 대신 다른 모델로 대체:

```
Agent(
  description: "Haiku code review",
  subagent_type: "general-purpose",
  model: "haiku",
  mode: "auto",
  prompt: <동일한 리뷰 프롬프트>
)
```

사용자에게 "Codex 대신 Haiku로 대체되었습니다"라고 알린다.

### Mode B: 순차 듀얼 퍼스펙티브 (서브에이전트 환경)

Agent 도구가 없으므로, 본인이 직접 두 번의 **구분된 리뷰 패스**를 수행한다.

#### Pass 1: 아키텍처/설계 관점 (Opus 시점)

다음에 집중하여 리뷰:
- 아키텍처 결정, 설계 패턴, SOLID 원칙
- 결합도/응집도, 추상화 수준
- 확장성, 유지보수성
- 타입 안전성, API 계약

Pass 1의 결과를 작성 완료한 후 Pass 2로 넘어간다.

#### Pass 2: 실용적 버그/런타임 관점 (Codex 시점)

**Pass 1과 의식적으로 다른 시각**을 취한다. Pass 1에서 이미 발견한 이슈라도, 이 관점에서 독립적으로 발견할 수 있는 것이면 별도로 기록한다:
- 구체적 버그, 엣지 케이스, 런타임 오류
- 성능 병목, 메모리 누수, 리소스 관리
- 보안 취약점, 인젝션, 인증 우회
- 실제 동작 시 발생할 수 있는 문제

두 패스 결과를 Step 3의 비교 기준에 따라 분류한다.

### 리뷰 프롬프트 템플릿

두 리뷰어에게 **동일한 프롬프트**를 제공하여 공정한 비교를 보장한다. 아래 템플릿에 수집한 컨텍스트를 채워서 사용:

```
당신은 시니어 코드 리뷰어입니다. 아래의 코드 변경사항을 독립적으로 리뷰해주세요.
다른 리뷰어도 동일한 코드를 리뷰하고 있지만, 그 리뷰어의 의견은 보지 못합니다.
자신만의 관점과 기준으로 솔직하게 리뷰하세요.

## 리뷰 대상

### 변경 파일 목록
[git diff --stat 결과]

### Diff
[git diff 전체 결과]

### 변경 파일 전체 내용
[각 변경 파일의 전체 내용]

## 리뷰 관점

자신만의 독립적인 기준으로 리뷰하되, 최소한 다음 영역은 포함:

1. **정확성 (Correctness)**: 버그, 논리 오류, 엣지 케이스, off-by-one
2. **설계 (Design)**: 아키텍처, 패턴, 결합도, 응집도, SOLID 원칙
3. **성능 (Performance)**: 불필요한 연산, 메모리 누수, N+1 문제, 블로킹 호출
4. **보안 (Security)**: 인젝션, 인증/인가, 민감 정보 노출, OWASP Top 10
5. **가독성 (Readability)**: 네이밍, 복잡도, 코드 구조, 매직 넘버
6. **테스트 (Testing)**: 테스트 커버리지, 테스트 품질, 엣지 케이스 테스트

위 영역 외에 본인이 중요하다고 판단하는 사항이 있다면 자유롭게 추가하라.

## 출력 형식

각 발견사항을 다음 형식으로 작성:

### [심각도: Critical/High/Medium/Low] 제목
- **파일**: `파일경로:라인번호`
- **설명**: 무엇이 문제이고 왜 문제인지
- **제안**: 구체적인 개선 방안 (가능하면 코드 포함)

마지막에 전체 요약과 승인 여부를 판단:
- **APPROVE**: 큰 문제 없음, 머지 가능
- **REQUEST_CHANGES**: 수정이 필요한 사항 있음
- **COMMENT**: 참고사항만 있음, 머지 차단하지 않음
```

## Step 3: 비교 & 토론

두 리뷰가 완료되면, 메인 에이전트(당신)가 결과를 비교 분석한다.

### 비교 분류

두 리뷰를 파일/이슈 단위로 대조하여 4가지로 분류:

1. **합의점 (Agreed)**: 두 리뷰어가 동일하거나 유사한 이슈를 지적 → **신뢰도 높음**. 이 항목들은 반드시 수정을 권고한다.
2. **Opus만 지적 (Opus Only)**: Opus가 발견했지만 Codex가 놓친 사항
3. **Codex만 지적 (Codex Only)**: Codex가 발견했지만 Opus가 놓친 사항
4. **상충 의견 (Disputed)**: 동일 이슈에 대해 두 리뷰어의 의견이 상반되는 경우

### 토론 및 판정

상충 의견이 있을 때, 다음 기준으로 어떤 의견이 더 타당한지 판단:

- 코드의 실제 동작과 일치하는가? (필요하면 코드를 직접 읽어 확인)
- 프로젝트 컨텍스트(프레임워크, 배포 환경)에 부합하는가?
- 일반적인 업계 모범 사례에 맞는가?
- YAGNI 원칙에 위배되지 않는가?

판단이 어려운 경우, 양쪽 의견을 모두 사용자에게 제시하고 결정을 맡긴다.

## Step 4: 리포트 생성

프로젝트 루트에 `.reviews/` 디렉토리를 생성하고, PR/MR 스타일 마크다운 리포트를 저장한다.

### 파일 경로 규칙

```
.reviews/
  YYYY-MM-DD_HH-MM-SS_<short-summary>.md
```

**타임스탬프는 반드시 실제 현재 시간을 사용한다.** `date` 명령이나 시스템 시간을 확인하여 정확한 시간을 기록할 것:

```bash
date '+%Y-%m-%d_%H-%M-%S'
```

예: `.reviews/2026-04-01_14-30-22_add-auth-middleware.md`

### 리포트 템플릿

```markdown
# Code Review Report

> **Date**: YYYY-MM-DD HH:MM:SS
> **Reviewers**: Claude Opus, OpenAI Codex
> **Verdict**: APPROVE | REQUEST_CHANGES | COMMENT
> **Files reviewed**: N files (+additions / -deletions)

## Summary

[전체 변경사항에 대한 간결한 요약 — 무엇이 왜 변경되었는지]

## Review Scope

| File | Changes |
|------|---------|
| `path/to/file.py` | +10 / -3 |

---

## Findings

### Agreed (Both reviewers)

> 두 리뷰어가 모두 지적한 사항. 신뢰도가 높은 항목.

#### [심각도] 제목
- **File**: `path:line`
- **Opus**: [Opus의 분석]
- **Codex**: [Codex의 분석]
- **Suggestion**: 구체적 개선안

---

### Opus Only

> Opus만 지적한 사항.

#### [심각도] 제목
- **File**: `path:line`
- **Analysis**: [분석 내용]
- **Suggestion**: 개선안

---

### Codex Only

> Codex만 지적한 사항.

#### [심각도] 제목
- **File**: `path:line`
- **Analysis**: [분석 내용]
- **Suggestion**: 개선안

---

### Disputed

> 두 리뷰어의 의견이 달랐던 사항과 토론 결과.

#### [심각도] 제목
- **File**: `path:line`
- **Opus says**: [Opus 의견]
- **Codex says**: [Codex 의견]
- **Resolution**: [토론 결과 및 최종 판단 근거]

---

## Proposed Changes

변경이 필요한 경우, 심각도 높은 순서로 나열:

- [ ] `file:line` — 설명
- [ ] `file:line` — 설명

## Raw Reviews

<details>
<summary>Opus Full Review</summary>

[Opus 리뷰 전문]

</details>

<details>
<summary>Codex Full Review</summary>

[Codex 리뷰 전문]

</details>
```

## Step 5: 사용자 승인

리포트 생성 후:

1. 리포트 파일 경로를 알려준다
2. 주요 발견사항을 **심각도 높은 순서**로 요약하여 보여준다 (Critical → High → Medium)
3. "Proposed Changes" 섹션의 변경 사항을 적용할지 사용자에게 묻는다
4. **사용자가 명시적으로 승인한 항목만** 코드에 반영한다
5. 변경 적용 후, 리포트의 체크리스트를 `[x]`로 업데이트한다

절대로 사용자 승인 없이 코드를 수정하지 않는다.

## 사용 예시

### 기본 사용
```
/dual-review
```
→ 작업 트리의 모든 변경사항 리뷰

### 특정 파일 리뷰
```
/dual-review src/auth.py src/middleware.py
```

### 특정 브랜치/커밋 기준
```
/dual-review --base main
```

### 최근 N개 커밋
```
/dual-review --commits 3
```

## 폴백 동작

- **Codex 사용 불가**: Codex 서비스가 응답하지 않거나 설치되어 있지 않은 경우, Opus 단독 리뷰 + Haiku 리뷰 (model: "haiku")로 대체한다. 사용자에게 폴백되었음을 알린다.
- **대규모 변경**: diff가 5000줄을 초과하는 경우, 파일 단위로 분할하여 리뷰한다. 각 파일 그룹에 대해 별도의 병렬 리뷰를 실행하고, 결과를 하나의 리포트로 합친다.
- `.reviews/` 디렉토리는 자동 생성되며, 프로젝트마다 독립적으로 관리된다.
