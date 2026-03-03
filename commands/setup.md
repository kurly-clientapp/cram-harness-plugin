---
description: "CRAM Harness V2 프로젝트 뼈대를 생성하고, 전역 규칙을 글로벌 CLAUDE.md에 주입합니다."
---

# /cram-harness:setup — 프로젝트 초기화

이 명령어가 실행되면 너는 다음 작업을 **순서대로** 수행해야 해:

---

## 1단계: Git 저장소 검증

아래 명령어를 실행하여 현재 디렉토리가 Git 저장소인지 확인한다:

```bash
git rev-parse --is-inside-work-tree
```

- **성공 시**: 다음 단계로 진행한다.
- **실패 시**: 아래 경고를 출력하되, 초기화를 **중단하지 않고** 계속 진행한다.

> ⚠️ 경고: 현재 디렉토리는 Git 저장소가 아닙니다. `commit-plan` 명령 사용 시 커밋 해시 추적이 비활성화됩니다. `git init`을 먼저 실행하는 것을 권장합니다.

---

## 2단계: 디렉토리 구조 생성

아래 디렉토리를 모두 생성한다 (이미 존재하면 건너뛴다):

- `rules/`
- `memory/drafts/`
- `plans/`
- `.harness_data/fine_tuning/`
- `.harness_data/rag_db/`

---

## 3단계: 파일 생성

아래 파일을 생성한다. **이미 존재하는 파일은 덮어쓰지 않는다** (데이터 보호):

### `rules/MAP.md`

```markdown
# 🗺️ Project Knowledge Map (Domain Routing)
<!-- 프로젝트 도메인별 지식 라우팅 맵을 여기에 작성하세요 -->
```

### `memory/MEMORY.md`

```markdown
# Episodic Memory Log
<!-- 에피소딕 메모리가 자동으로 기록됩니다. 직접 수정하지 마세요. -->
```

---

## 4단계: 글로벌 CLAUDE.md에 전역 규칙 주입 (핵심)

전역 규칙은 **프로젝트가 아닌 사용자 글로벌 설정**에 넣는다. 이렇게 하면 팀원에게 영향 없이 나만의 에이전트 행동 규칙을 적용할 수 있다.

**`~/.claude/CLAUDE.md` 파일이 이미 존재하면 덮어쓰지 않는다.** 존재하지 않을 때만 아래 내용으로 생성한다. 이미 존재하면, `## CRAM Protocol` 섹션이 있는지 확인하고 없을 때만 파일 끝에 Append한다.

```markdown
## 전역 가이드라인 (Global Harness)

모든 응답과 코드 생성 시 아래 원칙을 최우선으로 적용한다.

### 4대 개발 원칙
1. **단순성(YAGNI)** — 지금 필요하지 않은 것은 만들지 않는다.
2. **가역성(DIP)** — 의존성 역전 원칙을 통해 결정을 되돌릴 수 있게 설계한다.
3. **WWHP(Why→What→How→Proof!)** — 모든 작업은 '왜→무엇을→어떻게→증명' 순서를 따른다.
4. **OSL(Step-by-step)** — 복잡한 작업은 단계별로 분해하여 실행한다.

### 금지 패턴 (Cache Miss 유발)
- 세션 중 도구 추가/제거/파라미터 수정 (No Tool Shuffling)
- 시스템 프롬프트에 가변 정보(시간 등) 포함 금지
- 세션 중 모델 교체 (필요 시 서브에이전트 handoff 사용)

## CRAM Protocol — 운영 스펙

### Linter 에이전트 룰 (Self-Correction)
- Git 커밋 직전, 결과물이 전역 원칙(YAGNI, DIP)과 프로젝트 `rules/`에 위배되지 않는지 자가 검열해야 한다. 위배 시 계획을 수정한다.

### Cache-Safe Forking & Plan Mode
- Plan Mode 진입 시 프리픽스 유지를 위해 파일 수정을 금지한다 (Read-only 모드).
- 서브에이전트 spawn 시 부모와 동일한 시스템 프롬프트/도구셋을 유지한다.

### 컴팩션 전략
- 컨텍스트 80% 도달 시 현재 상태 5줄 요약 후 `/compact` 실행.
- 부모와 동일한 프리픽스를 유지한다 (Cache-Safe Forking).

### system-reminder 포맷
런타임 업데이트가 필요할 때는 아래 포맷을 사용한다:
`<system-reminder> [CATEGORY] {주제} | [INFO] {내용} </system-reminder>`
```

---

## 5단계: 프로젝트 CLAUDE.md 생성 (최소한)

**프로젝트 루트의 `CLAUDE.md` 파일이 이미 존재하면 덮어쓰지 않는다.** 존재하지 않을 때만 아래 최소 내용으로 생성한다:

```markdown
# 프로젝트 아키텍처 & 도메인

<!-- 프로젝트 고유 규칙을 여기에 작성하세요 -->
<!-- 전역 규칙(4대 원칙, CRAM 스펙)은 ~/.claude/CLAUDE.md에 있습니다 -->
```

---

## 6단계: 완료 보고

아래 메시지를 출력한다:

> ✅ CRAM Harness V2 프로젝트 초기화 완료!
>
> **글로벌 (나만 적용, 팀원 영향 없음):**
> - `~/.claude/CLAUDE.md` — 전역 규칙 주입 완료 (4대 원칙 + CRAM 스펙)
>
> **프로젝트 (팀 공유):**
> - `CLAUDE.md` — 프로젝트 고유 규칙 (최소 템플릿)
> - `rules/MAP.md` — 도메인 라우팅 맵
> - `memory/MEMORY.md` — 에피소딕 메모리 로그
> - `plans/` — 플랜 관리 디렉토리
> - `.harness_data/fine_tuning/` — 텔레메트리 데이터
>
> **참고:** Claude 네이티브 메모리(`~/.claude/projects/`)는 세션 연속성을, CRAM은 구조화된 에피소딕 러닝을 담당합니다. 두 시스템은 보완 관계입니다.
>
> 다음 단계: 작업을 마친 후 `/cram-harness:commit-plan`을 실행하여 에피소딕 메모리를 기록하세요.
