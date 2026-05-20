# 경량 하네스 + 대시보드 시스템 설계

**날짜:** 2026-05-20
**상태:** 설계 확정 (구현 전)

---

## 개요

oh-my-claudecode(OMC)의 무거운 토큰 사용량 문제를 해결하기 위한 경량 Claude Code 하네스 프레임워크. superpowers의 훈련된 프로세스(TDD, 인터뷰 기반 분석)와 GSD의 단계별 아티팩트 + 신선한 컨텍스트 방식을 결합한다. 브라우저 기반 웹 대시보드로 개발자뿐 아니라 PM, QA, 기획자도 진행 상황과 이력을 확인할 수 있다.

**참조 프로젝트:**
- [superpowers](https://github.com/obra/superpowers): TDD 방식, 인터뷰 기반 브레인스토밍, 스킬 시스템
- [GSD (get-shit-done)](https://github.com/gsd-build/get-shit-done): 단계별 아티팩트, 신선한 서브에이전트 컨텍스트

---

## 핵심 원칙

- **SSOT = Git 저장소**: 모든 변경 이력은 git 커밋으로 추적. 별도 DB 없음.
- **신선한 컨텍스트**: 각 단계는 독립된 서브에이전트로 실행 → 컨텍스트 오염 없음.
- **경량 우선**: autopilot, 복잡한 팀 파이프라인, 19개 에이전트 제거. 3개 에이전트 + 4개 슬래시 명령어만.
- **사람이 핵심 게이트 제어**: 완전 자율 실행 없음. 중요한 단계는 개발자가 CLI에서 승인.
- **비개발자 접근성**: 대시보드는 서버 없이 브라우저에서 직접 열림.

---

## 아키텍처

```
[ 사용자 ]
    │
    ▼
[ Harness Layer ]  ← CLAUDE.md + hooks + 4개 슬래시 명령어
    │  각 단계마다 아티팩트 생성 + git commit
    ▼
[ Artifact Layer ]  ← .artifacts/ 디렉토리 (git 추적)
    │  단계별 .md 파일 + index.json 갱신
    ▼
[ Dashboard Layer ]  ← 정적 HTML (브라우저에서 직접 열기)
                        index.json 읽어서 렌더링
```

---

## 하네스 설계

### 슬래시 명령어 (4개)

| 명령어 | 역할 | 에이전트 | 컨텍스트 |
|---|---|---|---|
| `/brainstorm` | 인터뷰 방식으로 요구사항 분석 → 설계 문서 생성 | 신선한 서브에이전트 | 없음 (첫 단계) |
| `/plan` | 설계 문서 기반 단계별 실행 계획 생성 | Planner (신선한 서브에이전트) | `requirements.md`, `design.md` 주입 |
| `/exec` | 계획 파일 기반 코드 구현 | Executor (신선한 서브에이전트) | `plan.md` 주입 |
| `/verify` | 구현 결과 검증 + 요구사항 충족 확인 | Verifier (신선한 서브에이전트) | `requirements.md`, `implementation.md` 주입 |

각 명령어는 직전 단계의 `.artifacts/*.md`를 **읽기 전용**으로 주입해서 컨텍스트를 이어받는다.

### `/brainstorm` 동작 방식

superpowers 스타일의 인터뷰:
1. 한 번에 하나씩 질문 (목적, 제약, 성공 기준 파악)
2. 가능하면 객관식 선택지 제공
3. 완료 시 `requirements.md` + `design.md` 생성 → git commit
4. 설계 확정 승인 게이트 트리거

### 에이전트 (3개)

| 에이전트 | 역할 | 제거하면 생기는 문제 |
|---|---|---|
| **Planner** | 요구사항 → 실행 가능한 단계별 계획 | 계획 없이 실행 시 컨텍스트 오염 |
| **Executor** | 계획에 따라 코드 구현 | 핵심 워크호스 |
| **Verifier** | 산출물이 요구사항 충족했는지 확인 | 품질 게이트 없음 |

OMC의 나머지 에이전트(explore, analyst, architect, debugger, code-reviewer 등)는 제거하거나 위 3개에 흡수.

---

## 워크플로우

```
/brainstorm
  → 인터뷰 (1문 1답)
  → requirements.md + design.md 생성
  → git commit
  → [승인 게이트 1: 설계 확정] ← 개발자 CLI 확인
        ↓
/plan
  → plan.md 생성
  → git commit
  → [승인 게이트 2: 계획 확정] ← 개발자 CLI 확인
        ↓
/exec
  → [승인 게이트 3: 보안 민감 코드] ← 자동 감지 시
  → implementation.md + git commit
        ↓
/verify
  → verification.md + git commit
  → [승인 게이트 4: 최종 검증] ← 개발자 CLI 확인
```

### 승인 게이트 (4개, hooks 기반)

| 게이트 | 트리거 | Hook 종류 |
|---|---|---|
| **설계 확정** | `/brainstorm` 완료 후 | `PostToolUse` |
| **계획 확정** | `/plan` 완료 후 | `PostToolUse` |
| **보안 민감 코드** | Executor가 민감 파일 수정 시 | `PreToolUse` |
| **최종 검증** | `/verify` 완료 후 | `PostToolUse` |

보안 감지 파일 패턴: `**/*auth*`, `**/*secret*`, `**/.env*`, `**/permission*`, `**/token*`, `**/credential*`

### 오류 처리

- `/exec` 실패 시: `implementation.md`에 오류 기록 → git commit → 개발자에게 알림. 재실행 시 이전 실패 아티팩트를 컨텍스트에 포함.
- `/verify` 실패 시: `verification.md`에 미충족 항목 기록 → 개발자가 `/exec` 재실행 여부 결정. 루프 횟수 제한 없음 (개발자 판단).
- 보안 게이트 거부 시: 해당 파일 수정 중단, 대안 구현 방법 제안.

---

## 아티팩트 & 상태 설계

### 디렉토리 구조

```
.artifacts/
├── index.json                          ← 대시보드가 읽는 유일한 파일
├── brainstorm-<timestamp>.md
├── requirements-<timestamp>.md
├── design-<timestamp>.md
├── plan-<timestamp>.md
├── implementation-<timestamp>.md
└── verification-<timestamp>.md
```

### `index.json` 스키마

```json
{
  "project": "프로젝트명",
  "stages": [
    {
      "stage": "brainstorm",
      "label": "요구사항 분석",
      "status": "approved",
      "timestamp": "2026-05-20T10:00:00Z",
      "commit": "abc1234",
      "artifact": "brainstorm-20260520T100000.md",
      "summary": "사용자가 요청한 기능 요약 한 줄"
    },
    {
      "stage": "plan",
      "label": "계획",
      "status": "pending",
      "timestamp": null,
      "commit": null,
      "artifact": null,
      "summary": null
    }
  ],
  "currentStage": "plan",
  "securityGates": [
    {
      "file": "src/auth/login.ts",
      "reason": "auth 패턴 감지",
      "approvedAt": "2026-05-20T11:00:00Z"
    }
  ]
}
```

### 쓰기 순서 (각 단계 완료 시)

1. `.artifacts/<stage>-<timestamp>.md` 작성
2. git commit (아티팩트 파일)
3. `index.json` 갱신
4. git commit (index.json)

---

## 대시보드 설계

### 기술 스택

- 순수 HTML + CSS + Vanilla JS (빌드 도구 없음)
- `.artifacts/index.json`을 `fetch()`로 읽어 렌더링
- 실행: 프로젝트 루트에서 `npx serve .` 후 `dashboard/index.html` 접근. `app.js`는 `../.artifacts/index.json`을 상대 경로로 fetch.

### 화면 구성

**상단: 현재 상태 패널**

```
[ 프로젝트명 ]          현재 단계: 계획          마지막 업데이트: 2026-05-20

  ✅ 요구사항 분석    ✅ 계획    🔄 구현    ⏳ 검증
  [승인됨]            [승인됨]   [진행중]   [대기]
```

**하단: 단계별 히스토리**

```
┌──────────────────────────────────────────────────────┐
│ ✅ 요구사항 분석  │  2026-05-20 10:00  │  abc1234   │
│ "로그인 기능 추가 — OAuth2 방식으로..."               │
│                                       [상세 보기 ▼]  │
├──────────────────────────────────────────────────────┤
│ ✅ 계획           │  2026-05-20 11:00  │  def5678   │
│ "3단계 구현 계획: auth → api → ui"                   │
│                                       [상세 보기 ▼]  │
├──────────────────────────────────────────────────────┤
│ 🔄 구현           │  진행중...                        │
└──────────────────────────────────────────────────────┘
```

**[상세 보기]** 클릭 시 해당 `.md` 파일 내용을 팝업으로 표시.

### 보안 승인 이력 패널

```
🔒 보안 승인 이력
  src/auth/login.ts  — auth 패턴 감지  — 승인됨 2026-05-20 11:00
```

### 비개발자 친화 요소

- 단계명 한국어 표시 (brainstorm → "요구사항 분석", exec → "구현")
- 상태는 이모지 + 텍스트 병행
- git 해시는 작게 표시 (개발자용 참조)
- 기술 용어 최소화

---

## CLAUDE.md 구성 (경량)

OMC 대비 제거:
- autopilot, ralph, ultrawork, team 파이프라인
- 19개 에이전트 카탈로그
- 복잡한 상태 관리 시스템 (`.omc/state/`)

유지:
- 4개 슬래시 명령어 정의
- 4개 hooks 정의 (승인 게이트)
- 아티팩트 쓰기 규칙
- 3개 에이전트 정의

---

## 파일 구조 (최종)

```
<프로젝트 루트>/
├── CLAUDE.md                    ← 하네스 설정
├── .artifacts/
│   ├── index.json
│   └── *.md
├── commands/
│   ├── brainstorm.md            ← /brainstorm 슬래시 명령어
│   ├── plan.md
│   ├── exec.md
│   └── verify.md
├── hooks/
│   ├── post-brainstorm.sh       ← 설계 확정 게이트
│   ├── post-plan.sh             ← 계획 확정 게이트
│   ├── pre-exec-security.sh     ← 보안 감지 게이트
│   └── post-verify.sh           ← 최종 검증 게이트
└── dashboard/
    ├── index.html
    ├── style.css
    └── app.js
```
