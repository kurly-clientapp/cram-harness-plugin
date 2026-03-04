---
description: "현재 플랜을 종료하고 에피소딕 메모리를 3-Phase 파이프라인으로 기록합니다."
---

# /cram-harness:commit-plan — Layer 7 에피소딕 메모리 커밋

이 명령어가 실행되면 너는 다음 작업을 **순서대로** 수행해야 해:

---

## 1단계: 컨텍스트 자동 추론 (Non-interactive)

사용자에게 질문하지 않는다. 대신 아래 **3-Tier 우선순위**에 따라 5개 필드를 자동으로 결정한다:

### 추론 우선순위

| 우선순위 | 소스 | 설명 |
|----------|------|------|
| ① **Manual** | 사용자가 커맨드 호출 시 명시한 값 | 예: `"도메인은 auth야"` 또는 자연어로 제공된 값 |
| ② **Inferred** | 세션 컨텍스트 + `.cram-harness/rules/MAP.md` 교차 검증 | 대화 맥락, 변경된 파일, 도메인 맵에서 추론 |
| ③ **Default** | 합리적 기본값 | 추론 불가 시 적용 (`severity`→`"none"`, `failed`→`"없음"`) |

### 필드별 추론 가이드

| 필드 | 추론 방법 |
|------|-----------|
| **DOMAIN** | 세션 중 수정/생성한 파일 경로에서 도메인 키워드 추출 → `.cram-harness/rules/MAP.md`에 정의된 도메인 맵과 대조하여 확정. MAP.md에 매핑이 없으면 파일 경로 기반 최선 추측 |
| **WHAT** | 세션에서 실제로 수행한 코드 변경·작성 내용을 1~2문장으로 요약 (에이전트 자동 생성) |
| **WHY** | 사용자가 세션 시작 시 던진 요청·목표에서 핵심 의도 추출. `.cram-harness/plans/` 폴더에 활성 플랜이 있으면 참조 |
| **FAILED** | 세션 대화에서 시행착오 지표 키워드를 스캔: **"아차", "다시", "에러", "롤백", "안 되네", "취소", "Wait", "Actually", "Rollback", "Error", "Doesn't work", "revert"** 등이 포함된 구간을 중점 분석하여 폐기된 접근법 요약. 해당 키워드가 없으면 `"없음"` |
| **SEVERITY** | 장애 대응·버그 핫픽스 맥락이면 `SEV-1`~`SEV-3` 추론, 일반 기능 개발·리팩토링이면 `"none"` |

---

## 2단계: 실행 모드 판별 (Normal vs Turbo)

추론이 완료되면, 사용자의 직전 명령(또는 세션 내 최근 지시)에서 **자율주행 트리거 키워드**를 확인한다:

**트리거 단어 목록** (하나라도 포함되면 Turbo 모드):
- 한국어: `"알아서"`, `"원스톱"`, `"자동으로"`, `"자동"`, `"한 번에"`, `"바로"`, `"즉시"`, `"지시대로"`
- 영어: `"autopilot"`, `"auto"`, `"one-stop"`, `"just do it"`

### Turbo 모드 (트리거 감지됨)

확인 절차 **없이** 즉시 3단계(Python 스크립트)를 실행한다. 완료 후 결과만 보고한다.

### Normal 모드 (트리거 미감지)

추론 결과를 아래 형태로 **한 번에 요약 출력**한 뒤, **1회만** 선택을 요청한다:

> 📋 **추론 결과 요약**
> | 필드 | 값 | 소스 |
> |------|-----|------|
> | DOMAIN | `auth` | MAP.md 매핑 |
> | WHAT | OAuth2 토큰 갱신 구현 | 세션 컨텍스트 |
> | WHY | 세션 만료 UX 개선 | 사용자 최초 발화 |
> | FAILED | Redis TTL 방식 | 키워드 스캔 |
> | SEVERITY | `none` | 기본값 |
>
> 어떻게 진행할까요?
> 1. **기록** — 바로 최종 기록합니다
> 2. **초안 저장** — 초안만 저장하고, 나중에 `/cram-harness:refine-plan`으로 디테일을 잡은 뒤 최종 기록합니다
> 3. **수정** — 필드를 수정한 뒤 다시 선택합니다

- **"기록"** 선택 시: 3단계(Python3 검증)로 진행한다.
- **"초안 저장"** 선택 시: 3단계(Python3 검증) 후 **4-A단계(초안 저장)**로 진행한다.
- **"수정"** 선택 시: 해당 필드만 교체 후 위 선택지를 다시 제시한다.

---

## 3단계: Python3 사전 검증

아래 명령어를 실행하여 Python3가 설치되어 있는지 확인한다:

```bash
command -v python3
```

- **성공 시**: 4단계로 진행한다.
- **실패 시**: 아래 메시지를 출력하고 **중단**한다.

> ❌ Python3가 설치되어 있지 않습니다. 에피소딕 메모리 기록에 Python3가 필요합니다.
> - macOS: 기본 탑재 (확인: `python3 --version`)
> - Linux/WSL: `sudo apt install python3` 또는 `sudo yum install python3`

---

## 4-A단계: 초안 저장 (Draft Save — "초안 저장" 선택 시에만 실행)

사용자가 **"초안 저장"**을 선택했을 때만 이 단계를 실행한다. 1~2단계에서 추론된 값을 `.cram-harness/memory/drafts/`에 마크다운 파일로 저장한다:

```bash
python3 -c "
import json, subprocess, datetime, os, sys, hashlib

# ── 입력값 ──
DOMAIN   = sys.argv[1]
WHAT     = sys.argv[2]
WHY      = sys.argv[3]
FAILED   = sys.argv[4]
SEVERITY = sys.argv[5] if len(sys.argv) > 5 else 'none'

# ── 메타데이터 수집 ──
NOW  = datetime.datetime.now(datetime.timezone.utc)
DATE = NOW.strftime('%Y-%m-%d')
TS   = NOW.isoformat()

try:
    COMMIT = subprocess.check_output(
        ['git', 'rev-parse', '--short', 'HEAD'],
        stderr=subprocess.DEVNULL
    ).decode().strip()
except Exception:
    COMMIT = 'uncommitted'

try:
    BRANCH = subprocess.check_output(
        ['git', 'rev-parse', '--abbrev-ref', 'HEAD'],
        stderr=subprocess.DEVNULL
    ).decode().strip()
except Exception:
    BRANCH = 'unknown'

raw = f'{TS}|{DOMAIN}|{COMMIT}|{WHAT}'
EPISODE_ID = hashlib.sha256(raw.encode()).hexdigest()[:12]

# ── 초안 저장 ──
os.makedirs('.cram-harness/memory/drafts', exist_ok=True)
draft_path = f'.cram-harness/memory/drafts/DRAFT-{EPISODE_ID}.md'
with open(draft_path, 'w', encoding='utf-8') as f:
    f.write(f'# 📝 Draft Episode: {EPISODE_ID}\n\n')
    f.write(f'<!-- status: draft -->\n')
    f.write(f'<!-- created: {TS} -->\n\n')
    f.write(f'| 필드 | 값 |\n')
    f.write(f'|------|-----|\n')
    f.write(f'| DOMAIN | {DOMAIN} |\n')
    f.write(f'| WHAT | {WHAT} |\n')
    f.write(f'| WHY | {WHY} |\n')
    f.write(f'| FAILED | {FAILED} |\n')
    f.write(f'| SEVERITY | {SEVERITY} |\n\n')
    f.write(f'## 메타데이터\n\n')
    f.write(f'- date: {DATE}\n')
    f.write(f'- branch: {BRANCH}\n')
    f.write(f'- commit: {COMMIT}\n')

print(f'\n📝 초안이 저장되었습니다!')
print(f'   경로: {draft_path}')
print(f'   에피소드 ID: {EPISODE_ID}')
print(f'\n   빌드 에러 수정, 디테일 확인 후 /cram-harness:refine-plan 으로 최종 기록하세요.')
print(f'   → FAILED, SEVERITY 등을 더 정확하게 보강할 수 있습니다.')
" "DOMAIN_VALUE" "WHAT_VALUE" "WHY_VALUE" "FAILED_VALUE" "SEVERITY_VALUE"
```

초안 저장 후 **5-A단계(초안 완료 보고)**로 진행한다.

---

## 4-B단계: 에피소딕 메모리 기록 (3-Phase Pipeline — "기록" 선택 또는 Turbo 모드)

사용자가 **"기록"**을 선택했거나 **Turbo 모드**일 때 이 단계를 실행한다. 1~2단계에서 확정된 값을 사용하여, 아래 Python3 스크립트를 실행한다. **반드시 이 코드를 그대로 사용**하고 포맷을 변경하지 마라:

```bash
python3 -c "
import json, subprocess, datetime, os, sys, hashlib

# ── 입력값 ──
DOMAIN   = sys.argv[1]
WHAT     = sys.argv[2]
WHY      = sys.argv[3]
FAILED   = sys.argv[4]
SEVERITY = sys.argv[5] if len(sys.argv) > 5 else 'none'

# ── 메타데이터 수집 ──
NOW  = datetime.datetime.now(datetime.timezone.utc)
DATE = NOW.strftime('%Y-%m-%d')
TS   = NOW.isoformat()

# Git 메타데이터
try:
    COMMIT = subprocess.check_output(
        ['git', 'rev-parse', '--short', 'HEAD'],
        stderr=subprocess.DEVNULL
    ).decode().strip()
except Exception:
    COMMIT = 'uncommitted'

try:
    BRANCH = subprocess.check_output(
        ['git', 'rev-parse', '--abbrev-ref', 'HEAD'],
        stderr=subprocess.DEVNULL
    ).decode().strip()
except Exception:
    BRANCH = 'unknown'

# 에피소드 고유 ID (SHA256 첫 12자)
raw = f'{TS}|{DOMAIN}|{COMMIT}|{WHAT}'
EPISODE_ID = hashlib.sha256(raw.encode()).hexdigest()[:12]

# ══════════════════════════════════════════════
# Phase 1: 에피소딕 메모리 적재 (즉시 실행)
# ══════════════════════════════════════════════

# 1-A) .cram-harness/memory/MEMORY.md — 사람이 읽기 좋은 포맷으로 Append
os.makedirs('.cram-harness/memory', exist_ok=True)
with open('.cram-harness/memory/MEMORY.md', 'a', encoding='utf-8') as f:
    f.write(f'\n- [{DATE}] [{EPISODE_ID}] [DOMAIN: {DOMAIN}] -> [COMMIT: {COMMIT}] ({BRANCH})\n')
    f.write(f'  - [WHAT] {WHAT}\n')
    f.write(f'  - [WHY] {WHY}\n')
    f.write(f'  - [FAILED] {FAILED}\n')
    if SEVERITY != 'none':
        f.write(f'  - [SEVERITY] {SEVERITY}\n')

# 1-B) .cram-harness/data/fine_tuning/experiences.jsonl — 파인튜닝용 JSONL Append
os.makedirs('.cram-harness/data/fine_tuning', exist_ok=True)
record = {
    'episode_id': EPISODE_ID,
    'timestamp': TS,
    'date': DATE,
    'domain': DOMAIN,
    'branch': BRANCH,
    'commit': COMMIT,
    'what': WHAT,
    'why': WHY,
    'failed': FAILED,
    'severity': SEVERITY
}
with open('.cram-harness/data/fine_tuning/experiences.jsonl', 'a', encoding='utf-8') as f:
    f.write(json.dumps(record, ensure_ascii=False) + '\n')

# 1-C) .cram-harness/data/rag_db/postmortem.yaml — RAG 검색용 YAML Append
os.makedirs('.cram-harness/data/rag_db', exist_ok=True)
with open('.cram-harness/data/rag_db/postmortem.yaml', 'a', encoding='utf-8') as f:
    f.write(f'---\n')
    f.write(f'episode_id: {EPISODE_ID}\n')
    f.write(f'timestamp: {TS}\n')
    f.write(f'domain: {DOMAIN}\n')
    f.write(f'branch: {BRANCH}\n')
    f.write(f'commit: {COMMIT}\n')
    f.write(f'severity: {SEVERITY}\n')
    f.write(f'what: \"{WHAT}\"\n')
    f.write(f'why: \"{WHY}\"\n')
    f.write(f'failed: \"{FAILED}\"\n')

print(f'\n✅ Layer 7 에피소딕 메모리 적재 완료! (episode: {EPISODE_ID})')
print(f'   Phase 1 ✓  MEMORY.md + JSONL + YAML')

# ══════════════════════════════════════════════
# Phase 2: 패턴 분석 (TODO — 향후 구현)
# ══════════════════════════════════════════════
# - experiences.jsonl을 주기적으로 읽어 도메인별 실패 패턴 집계
# - 반복 실패 도메인 자동 경고 (예: auth 도메인 3회 연속 실패)
# - .cram-harness/rules/MAP.md 자동 업데이트 제안

# ══════════════════════════════════════════════
# Phase 3: 파인튜닝 데이터셋 변환 (TODO — 향후 구현)
# ══════════════════════════════════════════════
# - experiences.jsonl → OpenAI fine-tuning JSONL 포맷 변환
# - system/user/assistant 턴 자동 생성
# - 도메인별 필터링 및 품질 점수 산출
" "DOMAIN_VALUE" "WHAT_VALUE" "WHY_VALUE" "FAILED_VALUE" "SEVERITY_VALUE"
```

**인자 매핑 규칙** (추론 또는 수동 확정된 값을 아래 위치에 대입):
- `DOMAIN_VALUE` → 추론 또는 수동 확정된 도메인명 (`.cram-harness/rules/MAP.md` 교차 검증 완료)
- `WHAT_VALUE` → 세션 컨텍스트에서 자동 생성한 작업 요약
- `WHY_VALUE` → 추론 또는 수동 확정된 핵심 의도 (사용자 최초 발화 + `.cram-harness/plans/` 참조)
- `FAILED_VALUE` → 키워드 스캐닝 기반 추론 또는 수동 제공 (없으면 `"없음"`)
- `SEVERITY_VALUE` → 맥락 기반 추론 또는 수동 제공 (없으면 `"none"`)

---

## 5단계: 완료 보고

### 5-A: 초안 저장 완료 보고 ("초안 저장" 선택 시)

> 📝 초안이 저장되었습니다!
>
> | 항목 | 값 |
> |------|-----|
> | 초안 경로 | `.cram-harness/memory/drafts/DRAFT-{EPISODE_ID}.md` |
> | 에피소드 ID | `{EPISODE_ID}` |
> | 커밋 | `{COMMIT}` |
> | 브랜치 | `{BRANCH}` |
>
> **다음 단계:**
> 1. 빌드 에러 수정, 테스트 확인 등 디테일 작업을 진행하세요
> 2. `/cram-harness:refine-plan` 실행 → 초안의 FAILED, SEVERITY 등을 보강하여 최종 기록합니다

### 5-B: 최종 기록 완료 보고 ("기록" 선택 또는 Turbo 모드)

스크립트 실행이 성공하면 아래 메시지를 출력한다:

> ✅ 플랜이 성공적으로 종료되었습니다!
>
> **Phase 1 적재 완료:**
> | 출력 | 경로 | 용도 |
> |------|------|------|
> | MEMORY.md | `.cram-harness/memory/MEMORY.md` | 사람 읽기용 에피소딕 메모리 |
> | JSONL | `.cram-harness/data/fine_tuning/experiences.jsonl` | 파인튜닝 텔레메트리 |
> | YAML | `.cram-harness/data/rag_db/postmortem.yaml` | RAG 검색용 포스트모템 |
>
> 에피소드 ID: `{EPISODE_ID}` · 커밋: `{COMMIT}` · 브랜치: `{BRANCH}`
