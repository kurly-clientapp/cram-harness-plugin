# CRAM Harness Plugin

**CRAM(Cache-Robust Architecture Memory)** 하네스 플러그인입니다.  
에이전트의 전역 행동 규칙을 글로벌 설정(`~/.claude/CLAUDE.md`)에 자동 주입하고, 작업 완료 시 에피소딕 메모리를 구조화된 포맷으로 기록합니다.

- **외부 스크립트 0개** — 모든 로직이 커맨드 마크다운 안에 인라인으로 포함
- **의존성 최소** — Python3만 필요 (macOS 기본 탑재)
- **크로스 플랫폼** — macOS, Linux, WSL 지원

## 왜 CRAM Harness인가? — Claude 네이티브 메모리와의 차이

Claude Code는 `~/.claude/projects/` 에 세션 JSONL과 메모리를 자체 관리합니다. 그렇다면 이 플러그인은 왜 필요할까요?

### Claude 네이티브 메모리의 한계

| 항목 | Claude 네이티브 (`~/.claude/projects/`) | CRAM Harness |
|------|----------------------------------------|--------------|
| **데이터 구조** | 세션 단위 raw JSONL · 비정형 | 도메인·원인·등급으로 **구조화된** 에피소드 |
| **출력 포맷** | 내부 전용 (Claude만 읽음) | **3중 포맷** — Markdown(사람) + JSONL(파인튜닝) + YAML(RAG) |
| **시행착오 추적** | 없음 | `FAILED` 필드 — 폐기된 접근법을 자동 스캔·기록 |
| **장애 등급** | 없음 | `SEVERITY` (SEV-1~3) — 포스트모템 분류 |
| **행동 제어** | 없음 | 4대 원칙 + 금지 패턴 주입 → 에이전트 행동 **하네스** |
| **도메인 라우팅** | 없음 | `rules/MAP.md` — 도메인별 지식 맵 교차 검증 |
| **팀 공유** | 불가 (로컬 전용) | 프로젝트 `memory/`, `rules/` → Git으로 팀 공유 가능 |
| **파인튜닝 파이프라인** | 없음 | `experiences.jsonl` → 파인튜닝 데이터셋 변환 (Phase 3) |

### 핵심 포인트

> **Claude 네이티브 메모리 = "무엇을 했는지" 기록 (세션 리플레이)**
>
> **CRAM Harness = "왜 실패했고, 무엇을 배웠는지" 구조화 (에피소딕 러닝)**

Claude의 네이티브 메모리는 세션을 **재현**하는 데 초점이 맞춰져 있습니다. CRAM은 세션에서 **학습**을 추출하여, 다음 세션·다른 팀원·파인튜닝에 활용할 수 있는 형태로 구조화합니다.

두 시스템은 **대체 관계가 아니라 보완 관계**입니다:
- Claude 네이티브 → 세션 연속성 (컨텍스트 이어가기)
- CRAM Harness → 지식 누적 (도메인 학습 + 행동 하네스)

---

## 플러그인 구조

```
cram-harness-plugin/
├── .claude-plugin/
│   └── plugin.json              # 플러그인 매니페스트 (v1.0.0)
├── commands/
│   ├── setup.md                 # /cram-harness:setup
│   └── commit-plan.md           # /cram-harness:commit-plan
└── README.md
```

> **공식 규격**: 플러그인 루트에 `.claude-plugin/plugin.json`이 반드시 존재해야 합니다.  
> 자세한 스펙은 [Plugins Reference](https://code.claude.com/docs/en/plugins-reference)를 참고하세요.

---

## Quick Start (CLI)

```bash
# 1. 플러그인 클론 (최초 1회)
git clone https://github.com/kyuhyeonKurly/cram-harness-plugin.git ~/plugins/cram-harness-plugin

# 2. 타겟 프로젝트에서 Claude Code 실행
cd /path/to/your-project
claude --plugin-dir ~/plugins/cram-harness-plugin

# 3. 세션 안에서 프로젝트 초기화
#    /cram-harness:setup

# 4. 작업 완료 후 에피소딕 메모리 기록
#    /cram-harness:commit-plan
```

---

## 설치 및 적용 방법

### 방법 1: 로컬 클론 + 직접 로딩 (개발/테스트 추천)

플러그인을 전역 설치 없이 로컬에서 바로 사용할 수 있습니다.

**Step 1.** 플러그인 클론

```bash
git clone https://github.com/kyuhyeonKurly/cram-harness-plugin.git ~/plugins/cram-harness-plugin
```

**Step 2.** 타겟 프로젝트에서 Claude Code 실행

```bash
cd ~/Documents/Company/your-target-project
claude --plugin-dir ~/plugins/cram-harness-plugin
```

또는 Agent SDK (Python):

```python
from claude_agent_sdk import query, ClaudeAgentOptions

options = ClaudeAgentOptions(
    plugins=[{"type": "local", "path": "~/plugins/cram-harness-plugin"}],
)

async for message in query(prompt="Hello", options=options):
    if message.type == "system" and message.subtype == "init":
        print("Plugins:", message.data.get("plugins"))
        print("Commands:", message.data.get("slash_commands"))
```

**Step 3.** 로딩 확인

세션 시작 시 init 메시지에 아래가 표시되면 성공:

```
✓ Plugin loaded: cram-harness
✓ Commands: cram-harness:setup, cram-harness:commit-plan
```

### 방법 2: 프로젝트 레포에 포함 (팀 전체 적용)

프로젝트 레포에 서브모듈로 포함하면 팀원 전체가 동일한 규칙을 사용합니다.

```bash
# 프로젝트 루트에서
git submodule add https://github.com/kyuhyeonKurly/cram-harness-plugin.git .plugins/cram-harness
```

Agent SDK에서 상대 경로로 로딩:

```python
options = ClaudeAgentOptions(
    plugins=[{"type": "local", "path": "./.plugins/cram-harness"}],
)
```

### 방법 3: CLI 글로벌 설치

```bash
# Claude Code 내에서
/plugin install cram-harness
```

설치 경로: `~/.claude/plugins/cram-harness/`

---

## 사용법

### 1. 프로젝트 초기화: `/cram-harness:setup`

새 프로젝트 디렉토리에서 실행합니다. 에이전트가 다음을 자동으로 수행합니다:

1. **Git 저장소 검증** — 비-Git 디렉토리면 경고 출력 (중단하지 않음)
2. **디렉토리 생성** — `rules/`, `memory/drafts/`, `plans/`, `.harness_data/fine_tuning/`, `.harness_data/rag_db/`
3. **파일 생성** — `rules/MAP.md`, `memory/MEMORY.md`
4. **글로벌 `~/.claude/CLAUDE.md`에 전역 규칙 주입** — 4대 원칙 + CRAM 운영 스펙 (나만 적용, 팀원 무영향)
5. **프로젝트 `CLAUDE.md` 최소 생성** — 프로젝트 고유 규칙 작성용 템플릿

#### 글로벌 vs 프로젝트 분리 구조

| 위치 | 내용 | 팀원 영향 |
|------|------|-----------|
| `~/.claude/CLAUDE.md` | 4대 원칙 + CRAM 스펙 (전역 규칙) | **없음** (내 로컬만) |
| 프로젝트 `CLAUDE.md` | 프로젝트 고유 규칙 템플릿 | 공유 가능 |
| 프로젝트 `memory/`, `rules/` | 에피소딕 메모리, 도메인 맵 | 내가 설정한 것만 |

`~/.claude/CLAUDE.md`에 주입되는 규칙:

| 구분 | 내용 |
|------|------|
| 4대 원칙 | YAGNI, DIP, WWHP, OSL |
| 금지 패턴 | No Tool Shuffling, 가변 정보 포함 금지, 모델 교체 금지 |
| Linter 룰 | Git 커밋 전 자가 검열 (Self-Correction) |
| Plan Mode | 프리픽스 유지를 위한 Read-only 모드 |
| 컴팩션 | 컨텍스트 80% 시 5줄 요약 + `/compact` |
| system-reminder | `[CATEGORY] {주제} \| [INFO] {내용}` 포맷 |
| Cache-Safe Forking | 서브에이전트 spawn 시 부모 프리픽스 유지, 모델 전환은 handoff |

### 2. 에피소딕 메모리 기록: `/cram-harness:commit-plan`

작업을 마친 뒤 실행합니다. **Layer 7 3-Phase 파이프라인**으로 에피소드를 기록합니다:

1. **자동 추론** — 세션 컨텍스트와 `rules/MAP.md`에서 5개 필드를 자동 추론합니다:

   | 필드 | 추론 소스 |
   |------|-----------|
   | DOMAIN | 변경된 파일 경로 + `rules/MAP.md` 교차 검증 |
   | WHAT | 세션 수행 내역 자동 요약 |
   | WHY | 사용자 최초 발화 + `plans/` 참조 |
   | FAILED | 시행착오 키워드 스캐닝 ("에러", "Rollback" 등) |
   | SEVERITY | 작업 맥락 기반 추론 |

   - **Normal 모드**: 추론 결과를 요약 표시 후 1회 확인
   - **Turbo 모드**: `"알아서"`, `"자동으로"`, `"원스톱"` 등 키워드 감지 시 확인 없이 즉시 적재

2. **Phase 1 적재** — 인라인 Python3 코드로 세 곳에 동시 기록:

   | 출력 | 경로 | 용도 |
   |------|------|------|
   | MEMORY.md | `memory/MEMORY.md` | 사람 읽기용 에피소딕 메모리 |
   | JSONL | `.harness_data/fine_tuning/experiences.jsonl` | 파인튜닝 텔레메트리 |
   | YAML | `.harness_data/rag_db/postmortem.yaml` | RAG 검색용 포스트모템 |

3. **Phase 2 (TODO)** — 패턴 분석: 도메인별 실패 패턴 집계 및 반복 실패 경고
4. **Phase 3 (TODO)** — 파인튜닝 데이터셋 변환: JSONL → OpenAI fine-tuning 포맷

**추가된 필드:**
- `episode_id` — SHA256 기반 고유 에피소드 ID (12자)
- `timestamp` — UTC ISO 8601 타임스탬프
- `branch` — 현재 Git 브랜치명
- `severity` — 장애 등급 (`SEV-1` ~ `SEV-3` 또는 `none`)

**기록 포맷 예시:**

```markdown
# memory/MEMORY.md
- [2026-02-25] [a1b2c3d4e5f6] [DOMAIN: auth] -> [COMMIT: a1b2c3d] (feat/oauth)
  - [WHAT] OAuth2 리프레시 토큰 자동 갱신 구현
  - [WHY] 사용자 세션 만료 시 재로그인 UX 개선
  - [FAILED] Redis TTL 기반 갱신 → 분산 환경에서 동기화 불가
  - [SEVERITY] SEV-2
```

```json
// .harness_data/fine_tuning/experiences.jsonl (한 줄)
{"episode_id": "a1b2c3d4e5f6", "timestamp": "2026-02-25T09:30:00+00:00", "date": "2026-02-25", "domain": "auth", "branch": "feat/oauth", "commit": "a1b2c3d", "what": "OAuth2 리프레시 토큰 자동 갱신 구현", "why": "사용자 세션 만료 시 재로그인 UX 개선", "failed": "Redis TTL 기반 갱신 → 분산 환경에서 동기화 불가", "severity": "SEV-2"}
```

```yaml
# .harness_data/rag_db/postmortem.yaml
---
episode_id: a1b2c3d4e5f6
timestamp: "2026-02-25T09:30:00+00:00"
domain: auth
branch: feat/oauth
commit: a1b2c3d
severity: SEV-2
what: "OAuth2 리프레시 토큰 자동 갱신 구현"
why: "사용자 세션 만료 시 재로그인 UX 개선"
failed: "Redis TTL 기반 갱신 → 분산 환경에서 동기화 불가"
```

---

---

## 실전 워크플로우 (Daily Routine)

```
┌─ 1. Plan ──────────────────────────────────────┐
│  Claude 세션 시작 → /cram-harness:setup (최초 1회) │
│  → 오늘 할 작업 플랜 수립                         │
├─ 2. Execute ───────────────────────────────────┤
│  코드 작성 / 디버깅 / 리팩터링                     │
│  (CRAM 규칙이 자동으로 에이전트 행동을 제어)        │
├─ 3. Commit ────────────────────────────────────┤
│  /cram-harness:commit-plan                      │
│  → 세션 컨텍스트에서 5개 필드 자동 추론        │
│  → Turbo 시 확인 스킵 / Normal 시 1회 확인     │
│  → Phase 1 자동 적재 (MEMORY + JSONL + YAML)     │
├─ 4. Wrap-up ───────────────────────────────────┤
│  컨텍스트 80% 차면 /compact                      │
│  다음 세션에서 memory/MEMORY.md 참조              │
└────────────────────────────────────────────────┘
```

> **기본 동작: 자동 추론**
>
> `/cram-harness:commit-plan`만 호출하면, 에이전트가 세션 컨텍스트를 분석하여 DOMAIN, WHAT, WHY, FAILED, SEVERITY를 **자동으로 추론**합니다.
> 추론 결과를 요약 표시하고, 확인 후 기록합니다.
>
> **Turbo 모드 (확인 스킵):**
>
> > "이 작업 커밋하고 commit-plan까지 **알아서** 실행해줘"
>
> `"알아서"`, `"원스톱"`, `"자동으로"` 등의 키워드가 포함되면 확인 절차 없이 즉시 적재합니다.
>
> **Manual Override (특정 필드만 수동 지정):**
>
> > "커밋하고 commit-plan 실행해줘. 도메인은 payment, 장애 등급은 SEV-1"
>
> 명시한 필드는 사용자 값을 우선 적용하고, 나머지는 자동 추론합니다.

---

## 트러블슈팅

### 플러그인이 로딩되지 않을 때

1. **경로 확인** — `.claude-plugin/plugin.json`이 있는 루트 디렉토리를 가리키고 있는지 확인
2. **plugin.json 검증** — `python3 -m json.tool .claude-plugin/plugin.json`으로 JSON 문법 확인
3. **파일 권한** — 플러그인 디렉토리가 읽기 가능한지 확인

### 커맨드가 작동하지 않을 때

1. **네임스페이스 사용** — 플러그인 커맨드는 `cram-harness:커맨드명` 형식이 필수
2. **init 메시지 확인** — `slash_commands`에 커맨드가 표시되는지 확인
3. **커맨드 파일 검증** — `commands/` 디렉토리 안에 `.md` 파일이 있는지 확인

### 경로 해석 문제

1. **작업 디렉토리 확인** — 상대 경로는 현재 작업 디렉토리 기준으로 해석됨
2. **절대 경로 사용** — 안정성을 위해 절대 경로 권장
3. **경로 정규화** — Python의 `pathlib.Path`를 사용하여 경로 구성

---

## 왜 skills/를 안 쓰나요?

Claude Code SDK에서 `skills/`는 에이전트가 **필요할 때 능동적으로 호출하는 온디맨드 도구**(예: DB 조회, API 호출)를 위한 디렉토리입니다. YAGNI, DIP 같은 **전역 행동 강령**을 `skills/`에 넣으면, 에이전트가 해당 스킬을 호출하지 않는 한 규칙을 전혀 인지하지 못합니다.

따라서 전역 규칙은 글로벌 설정인 `~/.claude/CLAUDE.md`에 주입하여, 어떤 프로젝트에서든 에이전트가 **무조건** 읽도록 하는 것이 정답입니다. 팀원에게 영향을 주지 않으면서 나만의 규칙을 적용할 수 있습니다.

## 의존성

| 도구 | 용도 | 설치 |
|------|------|------|
| Python3 | JSONL 생성 (에피소딕 메모리) | macOS: 기본 탑재 / Linux: `apt install python3` |
| Git | 커밋 해시 추출 (선택) | 미설치 시 `uncommitted`으로 대체 |

## 참고 자료

### 공식 문서

- [Plugins](https://code.claude.com/docs/en/plugins) — 플러그인 개발 가이드
- [Plugins Reference](https://code.claude.com/docs/en/plugins-reference) — 기술 스펙 및 스키마
- [Plugins in the SDK](https://platform.claude.com/docs/en/agent-sdk/plugins) — SDK 플러그인 로딩
- [Slash Commands (SDK)](https://platform.claude.com/docs/en/agent-sdk/slash-commands) — SDK 슬래시 커맨드
- [Subagents (SDK)](https://platform.claude.com/docs/en/agent-sdk/subagents) — 서브에이전트 연동

### 하네스 설계 참고

이 플러그인의 CRAM 하네스 구조를 설계할 때 참고한 자료입니다:

- [@trq212 — 에이전트 하네스 아키텍처 논의](https://x.com/trq212/status/2024574133011673516)
- [@koylanai — CRAM 패턴과 프리픽스 캐시 최적화 전략](https://x.com/koylanai/status/2025286163641118915)

## 라이선스

MIT
