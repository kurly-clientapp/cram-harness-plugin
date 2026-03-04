---
description: "저장된 초안을 검토·보강한 뒤 최종 에피소딕 메모리로 기록합니다."
---

# /cram-harness:refine-plan — 초안 보강 및 최종 기록

이 명령어가 실행되면 너는 다음 작업을 **순서대로** 수행해야 해:

---

## 1단계: 초안 목록 조회

`.cram-harness/memory/drafts/` 디렉토리에서 `DRAFT-*.md` 파일 목록을 조회한다:

```bash
ls -1t .cram-harness/memory/drafts/DRAFT-*.md 2>/dev/null
```

- **초안이 없을 때**: 아래 메시지를 출력하고 **중단**한다.

> 📂 저장된 초안이 없습니다.
> `/cram-harness:commit-plan` 실행 시 "초안 저장"을 선택하면 초안이 생성됩니다.

- **초안이 1개일 때**: 해당 초안을 자동 선택하고 2단계로 진행한다.
- **초안이 여러 개일 때**: 목록을 아래 형태로 출력하고 **1회만** 선택을 요청한다:

> 📂 **저장된 초안 목록**
>
> | # | 에피소드 ID | 생성일 | 파일 |
> |---|-------------|--------|------|
> | 1 | `a1b2c3d4e5f6` | 2026-03-03 | `DRAFT-a1b2c3d4e5f6.md` |
> | 2 | `x9y8z7w6v5u4` | 2026-03-02 | `DRAFT-x9y8z7w6v5u4.md` |
>
> 어떤 초안을 보강할까요? (번호 입력)

---

## 2단계: 초안 내용 표시 및 보강

선택된 초안 파일을 읽어 내용을 표시한다:

> 📝 **초안 내용 (에피소드: {EPISODE_ID})**
>
> | 필드 | 현재 값 |
> |------|---------|
> | DOMAIN | `auth` |
> | WHAT | OAuth2 토큰 갱신 구현 |
> | WHY | 세션 만료 UX 개선 |
> | FAILED | 없음 |
> | SEVERITY | `none` |
>
> **보강 안내:** 초안 저장 이후 빌드 에러, 테스트 실패 등을 경험했다면 아래 필드를 보강하세요:
> - **FAILED** — 시행착오, 폐기된 접근법, 발견된 에러를 구체적으로 기술
> - **SEVERITY** — 장애 등급이 있으면 `SEV-1`~`SEV-3`으로 지정
> - 그 외 필드도 수정 가능합니다
>
> 수정할 내용이 있으면 알려주세요. 없으면 "이대로 기록"이라고 해주세요.

사용자의 응답에 따라 분기한다:

- **"이대로 기록"** 또는 수정 없음 → 3단계로 진행
- **수정 요청** → 해당 필드를 교체한 뒤 수정된 내용을 다시 표시하고, `"이대로 기록할까요?"` 라고 **1회만** 확인한다. 승인 시 3단계로 진행.

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

## 4단계: 최종 에피소딕 메모리 기록 및 초안 정리

보강이 완료된 값을 사용하여, 아래 Python3 스크립트를 실행한다. **반드시 이 코드를 그대로 사용**하고 포맷을 변경하지 마라:

```bash
python3 -c "
import json, subprocess, datetime, os, sys, hashlib

# ── 입력값 (보강 완료된 값) ──
DOMAIN   = sys.argv[1]
WHAT     = sys.argv[2]
WHY      = sys.argv[3]
FAILED   = sys.argv[4]
SEVERITY = sys.argv[5] if len(sys.argv) > 5 else 'none'
DRAFT_FILE = sys.argv[6] if len(sys.argv) > 6 else ''

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

# ══════════════════════════════════════════════
# Phase 1: 에피소딕 메모리 적재
# ══════════════════════════════════════════════

# 1-A) MEMORY.md
os.makedirs('.cram-harness/memory', exist_ok=True)
with open('.cram-harness/memory/MEMORY.md', 'a', encoding='utf-8') as f:
    f.write(f'\n- [{DATE}] [{EPISODE_ID}] [DOMAIN: {DOMAIN}] -> [COMMIT: {COMMIT}] ({BRANCH})\n')
    f.write(f'  - [WHAT] {WHAT}\n')
    f.write(f'  - [WHY] {WHY}\n')
    f.write(f'  - [FAILED] {FAILED}\n')
    if SEVERITY != 'none':
        f.write(f'  - [SEVERITY] {SEVERITY}\n')

# 1-B) experiences.jsonl
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

# 1-C) postmortem.yaml
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

# ── 초안 파일 정리 ──
if DRAFT_FILE and os.path.exists(DRAFT_FILE):
    os.remove(DRAFT_FILE)
    print(f'   🗑️  초안 파일 삭제: {DRAFT_FILE}')

print(f'\n✅ 초안 보강 → 최종 에피소딕 메모리 적재 완료! (episode: {EPISODE_ID})')
print(f'   Phase 1 ✓  MEMORY.md + JSONL + YAML')
" "DOMAIN_VALUE" "WHAT_VALUE" "WHY_VALUE" "FAILED_VALUE" "SEVERITY_VALUE" "DRAFT_FILE_PATH"
```

**인자 매핑 규칙** (보강 완료된 값을 아래 위치에 대입):
- `DOMAIN_VALUE` → 보강된 도메인명
- `WHAT_VALUE` → 보강된 작업 요약
- `WHY_VALUE` → 보강된 핵심 의도
- `FAILED_VALUE` → **사용자가 보강한** 시행착오·에러 내용 (핵심 보강 대상)
- `SEVERITY_VALUE` → 보강된 장애 등급
- `DRAFT_FILE_PATH` → 정리할 초안 파일 경로 (예: `.cram-harness/memory/drafts/DRAFT-a1b2c3d4e5f6.md`)

---

## 5단계: 완료 보고

스크립트 실행이 성공하면 아래 메시지를 출력한다:

> ✅ 초안 보강 → 최종 기록 완료!
>
> **Phase 1 적재 완료:**
> | 출력 | 경로 | 용도 |
> |------|------|------|
> | MEMORY.md | `.cram-harness/memory/MEMORY.md` | 사람 읽기용 에피소딕 메모리 |
> | JSONL | `.cram-harness/data/fine_tuning/experiences.jsonl` | 파인튜닝 텔레메트리 |
> | YAML | `.cram-harness/data/rag_db/postmortem.yaml` | RAG 검색용 포스트모템 |
>
> 에피소드 ID: `{EPISODE_ID}` · 커밋: `{COMMIT}` · 브랜치: `{BRANCH}`
>
> 💡 **Tip:** 초안 → 보강 워크플로우를 통해 FAILED, SEVERITY 필드가 실제 빌드·테스트 경험을 반영하여 더 정확해집니다.
