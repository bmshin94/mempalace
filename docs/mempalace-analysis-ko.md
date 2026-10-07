# MemPalace 전수조사 분석 리포트 (한국어)

> 작성일: 2026-10-07
> 분석 대상: [`bmshin94/mempalace`](https://github.com/bmshin94/mempalace)
> 원본(upstream): [`MemPalace/mempalace`](https://github.com/MemPalace/mempalace)
> 분석 기준 버전: v3.10.0 / 라이선스: MIT

이 문서는 MemPalace 저장소를 폴더 단위까지 전수조사한 결과와,
설치·활용·수익화 관점의 분석을 한국어로 정리한 것입니다.

---

## 목차

1. [프로젝트 개요](#1-프로젝트-개요)
2. [쉬운 설명 — 비유로 이해하기](#2-쉬운-설명--비유로-이해하기)
3. [폴더별 전수조사 결과](#3-폴더별-전수조사-결과)
4. [설치 및 사용법](#4-설치-및-사용법)
5. [플러그인 / 스킬 / MCP 관계](#5-플러그인--스킬--mcp-관계)
6. [API 토큰 필요 여부](#6-api-토큰-필요-여부)
7. [GitHub에서 주목받는 이유](#7-github에서-주목받는-이유)
8. [로컬 에이전트 구축 활용도](#8-로컬-에이전트-구축-활용도)
9. [React / PHP 구현 가능성](#9-react--php-구현-가능성)
10. [수익화 아이디어](#10-수익화-아이디어)
11. [주의사항 및 리스크](#11-주의사항-및-리스크)
12. [참고 링크](#12-참고-링크)

---

## 1. 프로젝트 개요

### 한 줄 정의

**AI가 대화를 잊지 않도록, 사용자의 로컬 머신에만 저장하는 기억(메모리) 시스템.**

### 규모

| 항목 | 수치 |
| --- | --- |
| Python 파일 | 126개 |
| Python 코드 | 약 69,725줄 |
| 테스트 파일 | 177개 (커버리지 기준 85%, Windows 80%) |
| MCP 툴 | 45개 |
| 지원 저장소 백엔드 | 6종 |
| i18n 언어 | 14개 (한국어 `ko.json` 포함) |
| CHANGELOG 릴리즈 엔트리 | 18건 (v3.3.x ~ v3.10.0) |
| 참조된 최고 PR 번호 | #2594 |

### 해결하는 문제

Claude Code, Cursor, Codex 등 AI 코딩 도구는 컨텍스트 윈도우가 가득 차면
이전 대화를 잃고, Claude Code의 세션 기록(JSONL)은 30일 후 삭제된다.
어제의 결정, 설계 의도, 인물 정보가 전부 소실된다.

### 핵심 설계 원칙 (CLAUDE.md 기준)

| 원칙 | 내용 |
| --- | --- |
| **Verbatim always** | 요약·의역·손실 압축 금지. 원문 그대로 저장 |
| **Incremental only** | 초기 빌드 이후 append-only. 중단되어도 기존 팰리스 무손상 |
| **Entity-first** | 실명 기반 키, DOB/ID/문맥으로 동명이인 구분 |
| **Local-first** | 기본값으로 외부 API 0개. BYOK는 항상 명시적 옵트인 |
| **Performance budgets** | 훅 500ms 이하, 시작 주입 100ms 이하 |
| **Privacy by architecture** | 데이터를 보낼 코드 자체가 없음 |
| **Background everything** | 파일링·인덱싱은 훅으로 백그라운드 처리, 채팅창 토큰 0 |

### 팰리스 구조

```
WING (윙)   — 사람 / 프로젝트 단위
  └── ROOM (룸)   — 날짜 / 주제 단위
        └── DRAWER (서랍) — 원문 verbatim 텍스트

CLOSET (벽장) = AAAK 인덱스 — LLM이 스캔하는 압축 색인, 드로어를 가리킴
Knowledge Graph — ENTITY -[PREDICATE]-> ENTITY (valid_from / valid_to)
```

### AAAK 포맷 (`mempalace/dialect.py`)

```
헤더:   FILE_NUM|PRIMARY_ENTITY|DATE|TITLE
제텔:   ZID:ENTITIES|topic_keywords|"key_quote"|WEIGHT|EMOTIONS|FLAGS
터널:   T:ZID<->ZID|label
아크:   ARC:emotion->emotion->emotion
```

- 감정 코드: `vul`, `joy`, `fear`, `trust`, `grief`, `wonder`, `rage`, `love`,
  `hope`, `despair`, `peace`, `humor`, `tender`, `raw`, `doubt`, `relief`,
  `anx`, `exhaust`, `convict`, `passion`
- 플래그: `ORIGIN`, `CORE`, `SENSITIVE`, `PIVOT`, `GENESIS`, `DECISION`, `TECHNICAL`

> **중요:** 소스 주석에 명시된 대로 AAAK는 **무손실 압축이 아니다.**
> 구조화된 요약(색인) 레이어이며, 원문은 항상 드로어에 별도 보관된다.
> 공개 벤치마크 96.6%는 raw 모드 수치이고 AAAK 모드 수치가 아니다.

### 메모리 웨이크업 스택 (`mempalace/layers.py`)

| 레이어 | 토큰 | 내용 |
| --- | --- | --- |
| L0 Identity | ~100 | "나는 누구인가" — 항상 로드 |
| L1 Essential Story | ~500–800 | 팰리스의 핵심 순간 — 항상 로드 |
| L2 On-Demand | ~200–500/건 | 특정 주제·윙이 나올 때 로드 |
| L3 Deep Search | 무제한 | 전체 시맨틱 검색 |

웨이크업 총비용 L0+L1 = 약 600–900 토큰 (컨텍스트 95% 이상 여유 유지).

### 공개 벤치마크 (README / benchmarks)

**LongMemEval — 검색 리콜 (R@5, 500문항)**

| 모드 | R@5 | LLM 필요 |
| --- | --- | --- |
| Raw (시맨틱 검색, 휴리스틱·LLM 없음) | **96.6%** | 없음 |
| Hybrid v4 (홀드아웃 450문항) | **98.4%** | 없음 |
| Hybrid v4 + LLM 리랭크 (전체 500) | ≥99% | 임의의 capable 모델 |

**기타 벤치마크**

| 벤치마크 | 지표 | 점수 | 비고 |
| --- | --- | --- | --- |
| LoCoMo (session, top-10, no rerank) | R@10 | 60.3% | 1,986문항 |
| LoCoMo (hybrid v5, top-10, no rerank) | R@10 | 88.9% | 동일 세트 |
| ConvoMem (전체 카테고리) | 평균 리콜 | 92.9% | 250항목 |
| MemBench (ACL 2025) | R@5 | 80.3% | 8,500항목 |

전부 `benchmarks/BENCHMARKS.md`의 명령으로 재현 가능하며,
문항별 결과 JSONL이 `benchmarks/results_*`에 커밋되어 있다.

---

## 2. 쉬운 설명 — 비유로 이해하기

### 문제: AI는 "단기기억상실증 환자"

AI의 머릿속은 **화이트보드**다. 넓지만 꽉 차면 앞부분부터 지워지고,
다음 세션에는 깨끗하게 비워져 있다.

### 해결: AI에게 "영구 공책"을 쥐여주는 것

MemPalace는 화이트보드 옆에 영구 보관 공책을 두는 역할이다.

### 비유 1 — 도서관

```
도서관 전체        = 팰리스 (~/.mempalace)
"한국사 별관"      = WING   — 사람/프로젝트별 큰 구역
"조선시대 자료실"  = ROOM   — 날짜/주제별 방
서랍 1~3번         = DRAWER — 원문이 그대로 들어있는 곳
색인 카드함        = CLOSET (AAAK) — 사서가 보는 카드. 책 안 읽고 위치 파악
인물관계도 벽      = Knowledge Graph — 관계 + 유효기간
```

핵심 3가지:
1. **드로어** = 원본 그대로 보관, 요약 없음
2. **클로짓** = AI가 빠르게 훑어 "어느 서랍인지" 판단
3. **그래프** = 관계 + 언제부터 언제까지

### 비유 2 — 검색

```
기존 Ctrl+F:  정확한 단어를 알아야 함 → "결제" 검색 → 문서 500개
MemPalace:    "돈 받는 거 왜 갈아탔지" → 의미로 검색 → 정확한 드로어 1건
```

처리 과정:
1. 로컬 임베딩 모델로 질문을 벡터화
2. 벡터 유사도 검색(ChromaDB) + 키워드 검색(SQLite BM25) **동시 수행**
3. 윙/룸 필터, 시간 근접도, 키워드 부스팅으로 재랭킹
4. **원문 그대로** 반환

### 비유 3 — 훅은 "자동 녹음기"

대화가 끝나거나 컨텍스트 압축 직전에 훅이 조용히 발동해 서랍에 저장한다.
채팅창에는 아무것도 출력되지 않는다.

`MISSION.md`에 따르면 v3에서는 훅이 채팅창에서 실행되어 세션당 약 $1.13이
재전송 다이어리 블록으로 낭비됐고, v4에서 전부 백그라운드로 이동해 0원이 되었다.

### 비유 4 — 웨이크업은 "아침 브리핑"

`mempalace wake-up` 실행 시 L0+L1만 로드해 600–900 토큰으로
"우리가 무엇을 하고 있었는지"를 복구한다. 전부 읽지 않고 필요한 만큼만 꺼낸다.

### 비유 5 — 셰어드 브레인은 "단톡방 + 공유 드라이브"

```
       팰리스 허브 (mempalace serve)
        ▲        ▲         ▲
   맥북 Claude  윈도 Codex  노트북 Cursor
```

한 호스트가 팰리스의 writer lease를 보유하고 모든 클라이언트의 쓰기를 직렬화한다.
메모리(드로어/KG/다이어리)와 로그스트림(이벤트/아티팩트) 두 레이어를 제공한다.

| | 메모리 | 로그스트림 |
| --- | --- | --- |
| 담는 것 | 기억할 가치가 있는 지속 지식 | 에이전트 간 이동 중인 작업 |
| 접근 | 시맨틱 검색 | 구조화 필터 + 롱폴 |
| 예시 | 결정, 사실, 사람, 결과 | 위임, 응답, 패치, ack |

판단 기준: 다른 에이전트가 **행동**해야 하면 이벤트,
미래 세션이 **알아야** 하면 드로어.

---

## 3. 폴더별 전수조사 결과

### 루트

| 경로 | 역할 |
| --- | --- |
| `README.md` | 설치, 벤치마크, 백엔드 표, 사칭 사이트 경고 |
| `MISSION.md` | 원작자(Milla Jovovich)의 설계 배경 서사 |
| `CLAUDE.md` / `AGENTS.md`(심링크) | 에이전트용 프로젝트 지침 + 설계 원칙 |
| `ROADMAP.md` | 브랜치 모델(main/develop/release-*) 및 계획 |
| `CHANGELOG.md` | 약 157KB, 18개 릴리즈 엔트리 |
| `CONTRIBUTING.md` / `SECURITY.md` / `LICENSE` | 기여 가이드, 보안 정책, MIT |
| `pyproject.toml` | 패키지 메타, 엔트리포인트, extras |
| `Cargo.toml` / `Cargo.lock` | Rust 워크스페이스 |
| `Dockerfile` / `Dockerfile.gpu` / `docker-compose.yml` | CPU / CUDA 이미지, 컴포즈 |
| `mcp.json` / `.mcp.json` | MCP 서버 등록 (`command: mempalace-mcp`) |
| `.pre-commit-config.yaml` / `.python-version` | 린트 훅, Python 3.12 |

### `mempalace/` — Python 본체

| 경로 | 역할 |
| --- | --- |
| `mcp_server/` | MCP 서버 패키지. `protocol.py`, `runtime.py`, `http.py`, `schemas.py`, `tools_read/write/kg/diary/coord.py`, `_guards.py`, `_session.py` |
| `cli/` | CLI 패키지. `parser.py` + `cmd_init/mine/query/repair/serve/sync/coord/update.py`, `_hub.py` |
| `searcher/` | 하이브리드 검색. `candidates.py`, `ranking.py`, `sqlite_bm25.py`, `filters.py`, `query.py`, `render.py`, `cli_search.py` |
| `backends/` | 저장소 플러그인. `base.py`(추상), `chroma.py`, `sqlite_exact.py`, `rust_exact.py`, `milvus.py`, `qdrant.py`, `pgvector.py`, `registry.py`, `embedding_wrapper.py`, `_sidecar.py`, `_magic.py` |
| `palace/` | 윙/룸/드로어 운영 |
| `palace_graph.py` | 룸 트래버설 + 크로스윙 터널 |
| `knowledge_graph.py` | 시간축 엔티티-관계 그래프 (SQLite) |
| `dialect.py` | AAAK 압축 다이얼렉트 |
| `layers.py` | L0–L3 메모리 웨이크업 스택 |
| `miner.py` / `convo_miner.py` / `convo_scanner.py` / `format_miner.py` | 프로젝트 파일·대화 로그 채굴 |
| `sweeper.py` | 메시지 단위 드로어 생성 (멱등, 재개 가능) |
| `normalize.py` | 트랜스크립트 포맷 감지·정규화 |
| `entity_detector.py` / `entity_registry.py` / `entities.py` | 엔티티 자동 탐지, 저장, 동명이인 구분 |
| `hallways.py` | 엔티티 연관 그래프 |
| `embedding.py` | 임베딩 모델 (minilm / embeddinggemma / openai-compat) |
| `llm_client.py` / `llm_refine.py` / `closet_llm.py` | LLM 프로바이더 추상화(ollama/openai-compat/anthropic) 및 정제 |
| `logstream.py` / `logsync.py` / `tasks.py` / `hlc.py` / `replica.py` | 에이전트 조율, 하이브리드 논리시계, 복제 |
| `hub_client.py` / `server_registry.py` / `service.py` / `daemon.py` / `transport.py` | 허브 클라이언트, 데몬, 전송 |
| `hooks_cli.py` / `hook_shell.py` | 훅 관리 |
| `query_sanitizer.py` / `query_parser.py` | 프롬프트 오염 방지, 질의 파싱 |
| `repair.py` / `migrate.py` / `dedup.py` / `backups.py` / `encoding_repair.py` / `collision_scan.py` | 복구, 마이그레이션, 중복제거, 백업 |
| `fact_checker.py` / `dynamics.py` / `general_extractor.py` / `room_detector_local.py` | 모순 탐지, 동역학, 추출, 로컬 룸 감지 |
| `spellcheck.py` / `exporter.py` / `split_mega_files.py` / `onboarding.py` | 보조 기능 |
| `update_awareness.py` | 옵트인 릴리즈 확인 (PyPI만 접근, 기본 OFF) |
| `mcp_proxy.py` / `mcp_light_server.py` | `mempalace-mcp` / `mempalace-light-mcp` 엔트리포인트 |
| `sources/` | RFC 002 소스 어댑터 플러그인 seam (`base.py`, `registry.py`, `transforms.py`, `context.py`) |
| `i18n/` | 14개 언어 JSON (be, de, en, es, fr, hi, id, it, ja, **ko**, pt-br, ru, zh-CN, zh-TW) |
| `instructions/` | 에이전트용 지침 (`help/init/mine/search/status/shared_brain_rules.md`) |
| `data/` | 패키지 동반 데이터 |
| `config.py` | 설정 + 입력 검증 (`sanitize_name()`, `sanitize_content()`) |

### `crates/` — Rust 네이티브 엔진

| 경로 | 역할 |
| --- | --- |
| `mempalace-core/` | 네이티브 벡터 스캔 코어 |
| `mempalace-py/` | PyO3 바인딩 (→ `rust_exact` 백엔드) |
| `mempalace-cli/` | 독립 실행 `mempalace-native` CLI (`bench` 서브커맨드 포함) |

`rust_exact`는 `sqlite_exact`와 **동일한 `sqlite_exact.sqlite3` 파일**을 사용하므로
데이터 마이그레이션이 필요 없다. 복잡한 필터, 임베딩 반환 요청,
네이티브 확장 미설치 시 Python 백엔드로 폴백한다.

### 에이전트 통합 레이어

| 경로 | 역할 |
| --- | --- |
| `skills/` | 에이전트 스킬 3종 — `mempalace`(설치/운영), `mempalace-recall`(검색 우선), `mempalace-task`(로그스트림 위임) |
| `.claude-plugin/` | Claude Code 플러그인 — `plugin.json`, `marketplace.json`, 명령어 5개, 훅 3개, 스킬 3개 |
| `.cursor-plugin/` | Cursor IDE 플러그인 |
| `.codex-plugin/` | Codex CLI 플러그인 |
| `.antigravity-plugin/` | Antigravity 플러그인 (`rules/`, `skills/`) |
| `.dsh-plugin/` | DSH 플러그인 (JS — `autosave.js`, `palace.js`, `recall.js`, `transcript.js` + 테스트) |
| `.agents/plugins/` | 범용 에이전트 마켓플레이스 매니페스트 |
| `commands/` | 슬래시 명령어 마크다운 5종 |
| `rules/` | `mempalace-recall.mdc` (Cursor 룰) |
| `integrations/` | `openclaw/SKILL.md`, `shared/coordination-protocol.md`, `shared/recall-protocol.md` |
| `hooks/` | 자동저장 셸 스크립트 — `mempal_save_hook.sh`, `mempal_precompact_hook.sh`, `mempal_session_end_hook.sh` + `antigravity/`, `cursor/` 서브셋 (각 `install.sh`, `lib/common.sh`, `STDIN_SHAPE.md`) |

### 문서 / 품질 / 배포

| 경로 | 역할 |
| --- | --- |
| `docs/` | `CLOSETS.md`, **`HISTORY.md`(공개 정정·철회 기록)**, `RELEASING.md`, `authored-at.md`, `schema.sql`, `write-routing-policy.md`, `format-coverage.md`, `virtual-line-numbering.md`, `recovery/`, `rfcs/001~005` |
| `docs/rfcs/` | 001 저장소 백엔드 플러그인, 002 소스 어댑터, 003 에이전트 로그스트림, 004 복제 팰리스, 005 에이전트 신원 라우팅 |
| `website/` | VitePress 공식 문서 — `guide/`(17), `concepts/`(7), `reference/`(7) |
| `benchmarks/` | LongMemEval / LoCoMo / ConvoMem / MemBench 재현 스크립트 + 결과 JSONL + `lean_mempalace/`, `model_eval/` |
| `tests/` | 177 파일. `conftest.py`, `_backend_conformance.py`, `mcp/`, `benchmarks/` |
| `.github/workflows/` | `ci.yml`, `publish.yml`, `docker-publish.yml`, `deploy-docs.yml`, `release-native.yml`, `version-guard.yml` |
| `deploy/` | `docker-compose.server.yml`, `mempalace-server.service`, `server.env.example` |
| `examples/` | `basic_mining.py`, `convo_import.py`, `HOOKS_TUTORIAL.md`, `mcp_setup.md`, `gemini_cli_setup.md`, `antigravity/`, `cursor/` |
| `tools/` | `backup_claude_jsonls.sh`, `find_orphan_claude_jsonls.sh`, `render_jsonl.py`, `save.md` |
| `scripts/` | `backfill_authored_at.py`, `docker-smoke.sh`, `mempalace_repair_encoding.py` |
| `landing/` | 단일 랜딩 페이지 (`index.html`) |
| `assets/` | 로고 |
| `.devcontainer/` | VS Code dev container |

### 저장소 백엔드 비교 (README 표 기준)

| 백엔드 | 모드 | 설치 | 네임스페이스 | Lexical | 설정 |
| --- | --- | --- | :-: | :-: | --- |
| `chroma` (기본) | 로컬 (임베디드) | 번들 | – | ✓ | – |
| `sqlite_exact` | 로컬 (정확 NumPy) | 번들 | – | ✓ | – |
| `rust_exact` | 로컬 (네이티브 벡터) | wheel/컴파일 | – | ✓ | – |
| `milvus` | 로컬 Lite / 서버 | `mempalace[milvus]` | ✓ | ✓ | `MEMPALACE_MILVUS_URI` |
| `qdrant` | 서버 (REST) | 번들 | ✓ | ✓ | `MEMPALACE_QDRANT_URL` |
| `pgvector` | 서버 (Postgres) | `mempalace[pgvector]` | ✓ | ✓ | `MEMPALACE_PGVECTOR_DSN` |

선택 방법: `--backend <name>`, `MEMPALACE_BACKEND=<name>`,
또는 `config.json`의 `"backend": "<name>"`.

---

## 4. 설치 및 사용법

### 방법 1 — 에이전트 가이드 설치 (권장)

```bash
npx skills add MemPalace/mempalace
```

이후 코딩 에이전트에게 "MemPalace를 설치해줘"라고 요청하면
OS 감지 → 패키지 설치 → MCP 설정 → 토폴로지 선택(개인 로컬 / 셰어드브레인 허브 /
기존 허브 클라이언트)까지 안내한다.

### 방법 2 — CLI 직접 설치

```bash
uv tool install mempalace      # 권장: 격리 환경, PEP 668 회피
pipx install mempalace         # 동등한 대안

# venv 안에서 import mempalace가 필요한 경우만
python -m venv .venv && source .venv/bin/activate
pip install mempalace
```

글로벌 `pip install`은 피해야 한다. `chromadb`, `numpy`, `grpcio` 등
무거운 의존성이 전역 site-packages와 충돌하고, Debian/Ubuntu/Homebrew
Python에서는 PEP 668로 거부될 수 있다.

### 방법 3 — Docker

```bash
docker pull ghcr.io/mempalace/mempalace:latest   # amd64 + arm64

# MCP 서버 (stdio — JSON-RPC 때문에 -i 필수)
docker run -i --rm -v mempalace-data:/data ghcr.io/mempalace/mempalace

# CLI (마이닝은 읽기전용 마운트로 충분)
docker run --rm -v mempalace-data:/data -v /path/to/project:/work:ro \
  ghcr.io/mempalace/mempalace mine /work
```

MCP 클라이언트 등록 예시:

```json
{
  "mcpServers": {
    "mempalace": {
      "command": "docker",
      "args": [
        "run", "-i", "--rm",
        "-v", "mempalace-data:/data",
        "-v", "/absolute/path/to/.claude/projects:/transcripts:ro",
        "ghcr.io/mempalace/mempalace"
      ]
    }
  }
}
```

- `~`, `$HOME`은 모든 MCP 클라이언트가 확장하지 않으므로 절대경로를 사용한다.
- 이후 경로는 컨테이너 경로 기준이다 (`/transcripts`, `~/.claude/projects` 아님).
- 리눅스에서 이미지는 uid 1000으로 동작한다. 마운트 디렉터리가 `0700`이면
  `PermissionError: [Errno 13]`이 발생한다. `--user`로 우회하면 `/data` 소유자가
  uid 1000이라 팰리스 쓰기가 불가능해진다.
- 첫 임베딩 호출 시 모델을 `/data`로 다운로드한다 (minilm ~80MB, embeddinggemma ~300MB).

GPU 이미지는 x86_64 전용이다 (`onnxruntime-gpu`에 aarch64 Linux wheel 없음).

### 방법 4 — Claude Code 플러그인

```
/plugin marketplace add MemPalace/mempalace
/plugin install mempalace
```

설치되는 구성:
- 슬래시 명령어 5개 — `/mempalace-init`, `/mempalace-mine`, `/mempalace-search`,
  `/mempalace-status`, `/mempalace-help`
- 훅 3개 — Stop / PreCompact / SessionEnd
- MCP 서버 — `mempalace-mcp`
- 스킬 3개

### 기본 사용 흐름

```bash
# 0. 온보딩 — 임베딩 모델 선택
python -m mempalace.onboarding
#    embeddinggemma-300m : 다국어 100+개 (한국어 포함), ~300MB — 권장
#    all-MiniLM-L6-v2    : 영어 전용, ~30MB

# 1. 초기화 — 폴더 구조에서 룸 감지
mempalace init ~/projects/myapp

# 2. 마이닝
mempalace mine ~/projects/myapp                           # 프로젝트 파일
mempalace mine ~/.claude/projects/ --mode convos          # Claude Code 세션
mempalace mine ~/chats/ --mode convos --extract general   # 자동 분류 포함

# 3. 검색
mempalace search "왜 GraphQL로 바꿨더라"

# 4. 세션 컨텍스트 복구
mempalace wake-up

# 5. 상태 확인
mempalace status

# 6. 메시지 단위 정밀 리콜 (주기 실행, 멱등/재개 가능)
mempalace sweep ~/.claude/projects/
```

### 자동저장 훅 (가장 중요)

Claude Code 세션 기록(JSONL)은 30일 후 삭제된다. 훅을 걸지 않으면 영구 손실된다.

```bash
# 1. 기존 JSONL 백업
./tools/backup_claude_jsonls.sh

# 2. 훅 설치 (플러그인 설치 시 자동, 또는 hooks/ 수동 설치)

# 3. 과거 기록 백필
mempalace mine ~/.claude/projects/ --mode convos
```

Claude Code / Codex / Cursor / Antigravity 모두 훅을 지원하며,
Cursor는 세션 시작 리콜과 압축 전 트랜스크립트 스냅샷을 추가로 제공한다.

### CLI 명령어 전체 지도

| 그룹 | 명령어 |
| --- | --- |
| 기본 | `init`, `mine`, `sweep`, `search`, `wake-up`, `status`, `compress` |
| 서버/데몬 | `serve`, `mcp`, `daemon start/stop/status/jobs/wait` |
| 조율 | `logstream append/list/wait/watch/ack/sync`, `task create/launch`, `artifact put/get` |
| 유지보수 | `repair`, `migrate`, `migrate-wings`, `palace set-embedder`, `split` |
| 조회/설정 | `hallways`, `rules`, `instructions`, `hook run`, `sync` |
| 업데이트 | `update configure/check/plan` (기본 비활성) |

### 요구사항

- Python 3.9+
- 벡터 저장소 백엔드 (기본 ChromaDB)
- 임베딩 모델용 디스크 약 300MB
- Android/Termux 네이티브 설치는 미지원 (ChromaDB/ONNX에 Android wheel 없음)
  → Debian PRoot 컨테이너 사용 (`website/guide/termux.md`)

---

## 5. 플러그인 / 스킬 / MCP 관계

MemPalace는 **셋 중 하나가 아니라 다섯 개 레이어의 조합**이다.

```
5. 플러그인 레이어 — 설치 패키징
   .claude-plugin/ .cursor-plugin/ .codex-plugin/
   .antigravity-plugin/ .dsh-plugin/ .agents/
   → 명령어 + 훅 + MCP + 스킬을 한 번에 배포

4. 스킬 레이어 — 에이전트에게 사용법을 알려주는 마크다운 지침
   mempalace / mempalace-recall / mempalace-task
   → 실행 능력 없음

3. MCP 레이어 — 에이전트가 실제로 호출하는 45개 툴
   stdio / HTTP 두 전송 방식

2. 훅 레이어 — 백그라운드 자동저장 셸 스크립트
   Stop / PreCompact / SessionEnd / Wake

1. 코어 레이어 — Python 패키지 + Rust 엔진
   mempalace CLI, mempalace-mcp, mempalace-light-mcp, crates/
```

**핵심:** 스킬이나 플러그인만 설치해도 동작하지 않는다.
`skills/mempalace/SKILL.md`에 명시되어 있다:

> "Installing a skill does not by itself install the MemPalace CLI or MCP server."

올바른 순서: **Python 패키지 설치 → MCP 등록 → (훅 설치) → 스킬/플러그인**

### MCP 툴 45개 전체 목록

| 분류 | 툴 |
| --- | --- |
| 검색/조회 | `mempalace_search`, `mempalace_status`, `mempalace_get_taxonomy`, `mempalace_get_aaak_spec`, `mempalace_memories_filed_away` |
| 팰리스 구조 | `mempalace_list_wings`, `mempalace_list_rooms`, `mempalace_list_drawers`, `mempalace_get_drawer` |
| 드로어 CRUD | `mempalace_add_drawer`, `mempalace_update_drawer`, `mempalace_delete_drawer`, `mempalace_delete_by_source`, `mempalace_check_duplicate` |
| 네비게이션 | `mempalace_create_tunnel`, `mempalace_find_tunnels`, `mempalace_follow_tunnels`, `mempalace_list_tunnels`, `mempalace_delete_tunnel`, `mempalace_traverse`, `mempalace_list_hallways`, `mempalace_delete_hallway`, `mempalace_graph_stats` |
| 지식그래프 | `mempalace_kg_add`, `mempalace_kg_query`, `mempalace_kg_invalidate`, `mempalace_kg_supersede`, `mempalace_kg_timeline`, `mempalace_kg_stats` |
| 다이어리 | `mempalace_diary_read`, `mempalace_diary_write` |
| 조율 | `mempalace_event_append`, `mempalace_event_list`, `mempalace_event_wait`, `mempalace_event_ack`, `mempalace_task_create`, `mempalace_patch_submit`, `mempalace_artifact_put`, `mempalace_artifact_get`, `mempalace_mesh_peers` |
| 운영 | `mempalace_mine`, `mempalace_sync`, `mempalace_reconnect`, `mempalace_checkpoint`, `mempalace_hook_settings` |

### 클라이언트별 지원 현황

| 클라이언트 | 플러그인 | 스킬 | MCP | 훅 |
| --- | :-: | :-: | :-: | :-: |
| Claude Code | ✓ | ✓ | ✓ | Stop / PreCompact / SessionEnd |
| Cursor IDE | ✓ | ✓ | ✓ | Save / Wake / PreCompact |
| Codex CLI | ✓ | ✓ | ✓ | ✓ |
| Antigravity | ✓ | ✓ | ✓ | Save / Wake |
| Gemini CLI | – | – | ✓ | – |
| OpenClaw / DSH | ✓ | ✓ | ✓ | ✓ |
| AnythingLLM 등 MCP 호환 | – | – | ✓ | – |

---

## 6. API 토큰 필요 여부

**결론: 핵심 기능(저장·검색·웨이크업·마이닝)에는 API 토큰이 전혀 필요하지 않다.**

### 네트워크에 접근하는 모듈 전수조사

| 모듈 | 접근 대상 | 기본 상태 | 사용자 콘텐츠 전송 |
| --- | --- | --- | --- |
| `embedding.py` | HuggingFace (모델 1회 다운로드) | 필요 | 없음 |
| `llm_client.py` (ollama) | `localhost:11434` | 기본값 | 없음 (로컬) |
| `llm_client.py` (openai-compat) | 사용자 지정 URL | 옵트인 | 지정한 곳으로만 |
| `llm_client.py` (anthropic) | Anthropic Messages API | 옵트인 | 사용자가 켤 때만 |
| `closet_llm.py` / `llm_refine.py` | 위 `llm_client` 경유 | 옵트인 | 동일 |
| `hub_client.py` / `transport.py` | 사용자 허브 주소 | 옵트인 | 자신의 LAN/타넷 |
| `backends/qdrant.py` / `pgvector.py` | 사용자 지정 DB | 옵트인 | 자신의 서버 |
| `update_awareness.py` | `https://pypi.org/pypi/mempalace/json` | **기본 OFF** | 버전 문자열만 |
| ChromaDB 텔레메트리 | – | **명시적 비활성화** | 없음 |

### 로컬/외부 판별이 코드에 내장되어 있음

`mempalace/llm_client.py`:

```python
_LOCALHOST_HOSTS = frozenset({"localhost", "127.0.0.1", "::1"})

def _endpoint_is_local(url): ...
# LLMProvider.is_external_service() 가 이를 사용해 프라이버시 경고를 띄운다
```

### 외부 호출 제거 이력

CHANGELOG(GHSA-mrj5)에 따르면 `EntityRegistry.research()` /
`confirm_research()`의 Wikipedia 조회가 **게이팅이 아니라 삭제**되었다:

> "They are gone rather than merely gated, so local-first is a property of
> the code and not of a default argument."

또한 ChromaDB 텔레메트리는 침묵시키는 수준이 아니라 명시적으로 비활성화되었다 (GHSA-8h77).

### 토큰이 선택적으로 쓰이는 경우

| 기능 | 토큰 필요 | 로컬 대안 |
| --- | --- | --- |
| 저장 / 검색 / 웨이크업 / 마이닝 | 불필요 | – |
| 벤치마크 96.6% R@5 재현 | 불필요 | – |
| LLM 리랭크 (≥99%) | 선택 | Ollama 로컬 모델 |
| 엔티티 정제 (`llm_refine`) | 선택 | Ollama 로컬 모델 |
| 서버 임베딩 (GPU 활용) | 선택 | LM Studio / llama.cpp / vLLM (로컬·LAN) |

서버 임베딩은 `~/.mempalace/config.json`에 `embedding_model: "openai-compat"`,
`embedding_api_url`, `embedding_api_model` (필요시 `embedding_api_key`)로 설정한다.
각 키는 `MEMPALACE_EMBEDDING_API_*` 환경변수로 덮어쓸 수 있다.
전환 시 벡터 공간이 달라지므로 `mempalace repair rebuild-index`가 필요하다.

---

## 7. GitHub에서 주목받는 이유

> 이 세션은 포크 저장소에만 접근 권한이 있어 원본 저장소의 실제 스타 수는
> 확인하지 못했다. 아래는 저장소 내부 증거에 기반한 분석이다.

### 활동량 지표

| 지표 | 수치 |
| --- | --- |
| CHANGELOG 참조 최고 PR 번호 | #2594 |
| 릴리즈 엔트리 | 18건 (v3.3.x ~ v3.10.0) |
| 릴리즈 주기 | 약 2~4주 |
| Python 코드 | 69,725줄 / 126파일 |
| 테스트 파일 | 177개 (커버리지 85%) |
| 참조된 Discussion | #1388 |

PR 번호가 2,500을 넘는 것은 이슈·PR을 합쳐 상당한 규모의 커뮤니티
트래픽을 처리했음을 의미한다.

### 주목받는 구조적 이유 5가지

**1) 모든 AI 코딩 사용자가 겪는 문제를 정면으로 다룬다**

`MISSION.md`의 원작자 서술:

> "내 에이전트 Lumi가 계속 '안녕, 오늘 뭐 할까?' 하고 깨어나는데,
> 나는 그날 몇 시간이나 같이 일했다."

**2) 숫자가 명확하고 재현 가능하다**

`96.6% R@5 · API 0회 · 로컬`이 README 헤드라인이며,
재현 명령과 문항별 결과 JSONL이 저장소에 함께 커밋되어 있다.

```bash
uv run python benchmarks/longmemeval_bench.py /path/to/longmemeval_s_cleaned.json
```

**3) 프라이버시 서사가 강력하다**

"요약하지 않음 + 머신을 벗어나지 않음 + MIT 라이선스"는
개인과 기업 양쪽에 동시에 작동하는 포지셔닝이다.

**4) 과장된 주장을 스스로 철회했다 (신뢰도의 핵심)**

`docs/HISTORY.md` 기록:
- "+34% palace boost" 주장 → 철회 및 전 surface에서 제거
- "Haiku 리랭크로 100%" → 헤드라인에서 제거
  ("세 개의 오답을 들여다보고 도달한 수치 — teaching to the test")
- Mem0 ~85% / Zep ~85% 경쟁사 비교 → 출처 없음 및 지표 불일치로 삭제
- 검색 리콜(R@5/R@10)과 end-to-end QA 정확도를 같은 열에 놓은
  범주 오류 수정
- LoCoMo "100% R@10 with top-50 rerank" → 세션 수 19–32에 top_k=50이면
  구조적으로 전체 세션이 반환되므로 제거
- 커뮤니티 감사자 2명을 이름과 함께 명시적으로 감사

**5) 통합 범위가 넓다**

Claude Code / Cursor / Codex / Antigravity / Gemini CLI / OpenClaw / DSH /
AnythingLLM 등 어떤 도구 사용자든 진입 경로가 있다.

### 역설적 지표 — 브랜드 스쿼팅 발생

README 최상단 CAUTION:

> `mempalace.tech`, `.net`, 기타 `.com` 변종은 사칭이며 멀웨어를 배포할 수 있다.
> 공식 출처는 GitHub 저장소, PyPI 패키지, mempalaceofficial.com 세 곳뿐이다.

사칭 도메인이 생길 정도의 인지도가 확보되었다는 간접 증거다.

---

## 8. 로컬 에이전트 구축 활용도

**평가: 매우 높음.** 두 가지 방향으로 활용 가능하다.

### 방향 A — 부품으로 직접 사용

```python
from mempalace.searcher import search
from mempalace.palace import get_collection
from mempalace.knowledge_graph import KnowledgeGraph
```

즉시 확보되는 기능:
- 장기기억 — 6개 백엔드 중 선택
- 하이브리드 검색 — 벡터 + BM25 + 랭킹 구현 완료
- 시간축 지식그래프 — `valid_from` / `valid_to` 보유
- MCP 서버 — stdio/HTTP, 45개 툴
- 멀티에이전트 조율 — logstream(이벤트) + artifact(산출물) + task(위임)
- 에이전트별 윙/다이어리 — 런타임 발견, 시스템 프롬프트 비대화 없음
- 프롬프트 인젝션 방어 — `query_sanitizer.py`
- 14개 언어 i18n

### 방향 B — 설계 교재로 활용

| 학습 주제 | 참고 위치 | 가치 |
| --- | --- | --- |
| 플러그인 아키텍처 | `backends/base.py`, `registry.py`, entry_points | 벤더 종속 없는 추상화 실전 예제 |
| RFC 기반 설계 | `docs/rfcs/001~005` | 대형 기능을 문서로 선설계 |
| 훅 시스템 | `hooks/`, `hook_shell.py` | 백그라운드 자동화 패턴 |
| MCP 서버 구현 | `mcp_server/` (protocol/runtime/transport 분리) | MCP 구조 설계 |
| 동시성 제어 | writer lease, mine lock, WAL | 멀티프로세스 DB 공유 |
| 점진적 마이그레이션 | `migrate.py`, `repair.py` | 데이터 무손상 스키마 변경 |
| Python→Rust 전환 | `crates/` + PyO3 + 폴백 어댑터 | 안전한 성능 재작성 |
| 분산 복제 | `replica.py`, `hlc.py`, `logsync.py` | 멀티 디바이스 동기화 |
| 정직한 벤치마킹 | `benchmarks/BENCHMARKS.md`, `docs/HISTORY.md` | 홀드아웃, teaching-to-the-test 경계 |

### 권장 스택 구성

```
내 에이전트 (LangGraph / 직접 구현)
        │ MCP (45 tools)
        ▼
MemPalace — 장기기억 · 검색 · 지식그래프 · 에이전트 조율
        │
        ▼
Ollama (추론) + 로컬 임베딩  → 완전 오프라인, 토큰 0
```

### 주의사항

- Beta 단계(`Development Status :: 4 - Beta`) — 프로덕션은 버전 핀 고정 권장
- 의존성이 무겁다 (chromadb, numpy, grpcio, onnxruntime)
  → 경량 구성은 `sqlite_exact` 백엔드 + `mempalace-light-mcp` 활용
- Android 네이티브 미지원

---

## 9. React / PHP 구현 가능성

### React — 프론트엔드/확장에는 최적, 엔진 재작성은 비권장

#### 권장: React 기반 팰리스 탐색기

```
React / Next.js 팰리스 탐색기
  윙 트리뷰 · 룸 캘린더 · 드로어 뷰어(verbatim)
  검색 UI · 지식그래프 시각화(D3/Cytoscape)
  대시보드 · logstream 실시간 모니터
        │ HTTP
        ▼
MemPalace Python 엔진 (mempalace serve --port 8765)
```

구현 근거:
- `mempalace serve --host 127.0.0.1 --port 8765` — HTTP 서버가 이미 존재
- `mempalace/mcp_server/http.py` — HTTP 전송 구현 완료
- `deploy/docker-compose.server.yml` — 서버 배포 템플릿 제공

Tauri(용량 작음, Rust — `crates/` 재활용 가능) 또는 Electron으로 감싸면
데스크톱 앱이 된다. **엔진 코드 재작성 0줄.**

#### 비권장: Node/TypeScript 엔진 전체 재작성

| 항목 | 난이도 | 비고 |
| --- | --- | --- |
| 팰리스 CRUD | 낮음 | SQLite로 충분 |
| 임베딩 | 중간 | `transformers.js` / ONNX Runtime Node |
| 벡터 검색 | 중간 | `hnswlib-node`, LanceDB JS |
| BM25 | 낮음 | SQLite FTS5 |
| 마이너 / 노멀라이저 | 높음 | 포맷 처리 로직이 방대 |
| 지식그래프 | 낮음 | SQLite |
| MCP 서버 | 낮음 | TS SDK 존재 |
| 벤치마크 96.6% 재현 | 매우 높음 | 튜닝 노하우가 핵심 |

69,725줄 + 177개 테스트로 검증된 엔진을 재작성하는 ROI가 낮다.

### PHP — 프레젠테이션 레이어 전용

#### 가능

- 웹 대시보드/뷰어 — `mempalace serve` HTTP API를 호출하는 Laravel/Symfony 앱
- 팀 관리 패널 — 권한, 사용자 관리, 감사 로그
- 기존 PHP 시스템 통합 — 사내 위키·CRM에 MemPalace 검색 연동
- CMS 플러그인 — WordPress 사내 지식 검색

#### 부적합

| 필요 기능 | PHP 생태계 |
| --- | --- |
| 로컬 임베딩 (ONNX/Transformer) | 성숙한 라이브러리 없음 |
| 벡터 검색 (HNSW) | 네이티브 구현 사실상 없음 |
| 상주 프로세스 (데몬/롱폴링) | 가능하나 전통적 실행 모델과 불일치 |
| NumPy급 수치 연산 | 없음 |

### 권장 최종 아키텍처

```
프론트엔드 (React / Next.js)
  팰리스 탐색기 · 그래프 시각화 · 검색 UI
        │ REST / MCP over HTTP
        ▼
PHP 백엔드 (선택 — 기존 시스템이 있을 때)
  인증 · 권한 · 감사로그 · 팀 관리
        │ HTTP
        ▼
MemPalace (Python + Rust) — 수정하지 않음
  mempalace serve --port 8765
        │
        ▼
Ollama + 로컬 임베딩 — 완전 오프라인
```

---

## 10. 수익화 아이디어

### 전제 조건

| 항목 | 내용 |
| --- | --- |
| 라이선스 | MIT — 상업적 이용·수정·재배포·클로즈드소스 파생 자유 |
| upstream이 거부하는 PR | 사용자 콘텐츠 요약, 클라우드 저장/동기화, 텔레메트리/분석, 코어 메모리에 API 키 요구, verbatim 우회 |
| 전략적 함의 | 거부되는 기능이 곧 유료화 포인트 → **애드온 / 별도 레포 / 포크 / 서비스 레이어**로 구현 |
| 의무 | MIT 저작권 고지 유지 |

### 티어 1 — 즉시 착수 가능 (낮은 리스크)

#### 아이디어 1. 기업 온프레미스 도입 컨설팅 / SI

| 항목 | 내용 |
| --- | --- |
| 가격 | 초기 구축 500만~3,000만원 + 유지보수 월 100만~500만원 |
| 타겟 | 금융, 의료, 법률, 공공, 방산, 게임사 — 데이터 외부 유출 금지 조직 |
| 착수 | 즉시 (코드 수정 불필요) |
| 난이도 | 낮음 |

판매 논리: "AI 코딩을 도입하고 싶지만 코드가 외부로 나가면 안 된다"는
요구에 대해, MemPalace(로컬 전용) + Ollama(로컬 추론)로
완전 격리된 AI 개발 환경을 패키지로 제공한다.

서비스 구성: 환경 진단 → Docker/systemd 배포(`deploy/`) →
사내 마이닝 파이프라인 구축 → 팀 온보딩 교육 →
사내 룰셋 커스터마이징(`rules/`, `mempalace/instructions/`) → 유지보수 계약.

차별 자산: **로컬 전용임을 코드 수준에서 증명하는 보안 감사 리포트.**

#### 아이디어 2. 교육 콘텐츠 (강의 / 전자책 / 유튜브)

| 항목 | 내용 |
| --- | --- |
| 가격 | 강의 15만~50만원, 전자책 2만~5만원, 멤버십 월 2만~5만원 |
| 타겟 | AI 코딩 입문~중급 개발자, 1인 개발자 |
| 착수 | 2~4주 |
| 난이도 | 낮음 |

커리큘럼 초안:
1. AI가 잊는 이유 — 컨텍스트 윈도우 원리
2. MemPalace 설치와 첫 팰리스
3. 훅 설정과 자동저장 — 30일 삭제 대응
4. 검색 제대로 쓰기 — 윙/룸 스코핑
5. Ollama 연동으로 토큰 0원 만들기
6. 셰어드 브레인 — 여러 에이전트의 기억 공유
7. 로컬 에이전트 아키텍처 설계
8. (보너스) 플러그인 아키텍처 코드 리딩

수익 퍼널: 유튜브(무료 유입) → 전자책(저가) → 강의(고가) → 컨설팅(최고가).

#### 아이디어 3. 한국 시장 현지화 패키지

| 항목 | 내용 |
| --- | --- |
| 가격 | 월 2만~10만원 구독 또는 연 라이선스 |
| 타겟 | 한국 개발팀, 스타트업 |
| 착수 | 4~8주 |
| 난이도 | 낮음~중간 |

`ko.json`이 존재하지만 현지화는 번역 이상의 작업이다:

| 기능 | 내용 |
| --- | --- |
| 한국어 엔티티 인식 | 조사 처리("철수가/철수를/철수는"), 존칭, 직함 |
| 한국식 날짜 | "지난주 화요일", "담달 초", 음력, 공휴일 |
| 협업 도구 마이닝 | 카카오워크, 잔디, 라인웍스, 네이버웍스 어댑터 |
| 문서 포맷 | HWP / HWPX(한글) 마이너 |
| 조직도 그래프 | 사원→대리→과장→차장→부장 직급 체계 |
| 임베딩 튜닝 | KoSimCSE, BGE-m3-ko 등 벤치마킹 |

기술 근거: `mempalace/sources/`의 RFC 002 소스 어댑터 seam을 통해
별도 패키지로 깔끔히 분리 가능하다.

```toml
[project.entry-points."mempalace.sources"]
kakaowork = "mempalace_source_kakaowork:KakaoWorkAdapter"
```

방어막: 글로벌 도구가 한국어 조사·존칭·HWP를 다루지 않는다는 점이 진입장벽이 된다.

### 티어 2 — 제품화 (중간 리스크, 높은 업사이드)

#### 아이디어 4. GUI 데스크톱 앱 "팰리스 익스플로러" — 최우선 추천

| 항목 | 내용 |
| --- | --- |
| 가격 | 무료(기본) + Pro 1회 $49~99 또는 월 $5~9 |
| 타겟 | MemPalace 사용자 전체 |
| 착수 | MVP 6주, 제품화 2~3개월 |
| 난이도 | 중간 |

근거: 팰리스 데이터는 사용자의 기록 전체인데, 현재 열람 수단은 CLI뿐이다.
"내 기억을 눈으로 보고 싶다"는 수요가 구조적으로 보장된다.

무료 / Pro 분리:

| 기능 | 무료 | Pro |
| --- | :-: | :-: |
| 윙/룸/드로어 탐색 | ✓ | ✓ |
| 검색 | ✓ | ✓ |
| 지식그래프 시각화 | 기본 | 고급 (필터, 시간축 애니메이션) |
| 타임라인 뷰 | – | ✓ |
| 멀티 팰리스 관리 | – | ✓ |
| 암호화 백업/복원 | – | ✓ |
| 팀 허브 대시보드 | – | ✓ |
| 플러그인 SDK | – | ✓ |
| 자동 정리 제안 | – | ✓ |

기술: React + Tauri (또는 Electron) → `mempalace serve` HTTP API 호출.
엔진 재작성 없음.

#### 아이디어 5. 팀 셰어드 브레인 매니지드 호스팅

| 항목 | 내용 |
| --- | --- |
| 가격 | 시트당 월 $15~40 (5인 팀 월 $75~200) |
| 타겟 | 10~100명 개발팀, 에이전트 플릿 운영사 |
| 착수 | 3~4개월 |
| 난이도 | 중간~높음 |

판매 대상은 기능이 아니라 **운영 부담 제거**다:
서버 프로비저닝, TLS, 타넷/VPN, 백업 전략, SSO/RBAC,
무중단 업그레이드, SLA, 감사 로그 / 규정준수 리포트.

차별화: **제로 지식(zero-knowledge) 호스팅.**
클라이언트에서 암호화하여 업로드하고 서버는 복호화 키를 보유하지 않는다.
프로젝트 철학("데이터가 머신을 벗어나지 않는다")을 배신하지 않는 유일한 호스팅 형태다.

기술 제약: `mempalace serve`는 한 프로세스가 writer lease를 독점한다
(README: "never point two server processes at the same palace").
멀티테넌트는 **테넌트별 프로세스 격리** 설계가 필수다.

#### 아이디어 6. 도메인 특화 소스 어댑터 / 룰셋

| 항목 | 내용 |
| --- | --- |
| 가격 | 어댑터당 연 $200~2,000 (B2B) |
| 타겟 | 의료, 법률, 금융, 연구, 제조 |
| 착수 | 도메인당 2~3개월 |
| 난이도 | 중간 |

| 도메인 | 데이터 소스 | 특수 요건 |
| --- | --- | --- |
| 의료 | EMR 노트, 논문, 진료기록 | 비식별화 파이프라인 필수 |
| 법률 | 판례, 계약서, 사건기록 | 인용 추적 그래프 |
| 금융 | 리서치 리포트, 실적발표, 내부메모 | 감사 추적 |
| 연구 | LaTeX, Jupyter, 실험로그 | 재현성 추적 |
| 제조 | 설비로그, 품질보고서, SOP | 장애 이력 검색 |

각 도메인은 외부로 보낼 수 없는 "검색되지 않는 데이터"를 보유하고 있어
로컬 전용이 필수 요건이 된다.

### 티어 3 — 장기 / 고위험

#### 아이디어 7. E2E 암호화 백업·동기화 SaaS

| 항목 | 내용 |
| --- | --- |
| 가격 | 월 $3~10 (용량별) |
| 착수 | 4~6개월 |
| 난이도 | 높음 |

`replica.py`, `hlc.py`(하이브리드 논리시계), `logsync.py`로 복제 기반이 존재한다.
필수 조건: 클라이언트 사이드 암호화 100%. 미충족 시 프로젝트 철학 배신으로
커뮤니티 반발을 초래한다.

#### 아이디어 8. "CTO 기억 아카이브" (B2B 니치)

| 항목 | 내용 |
| --- | --- |
| 가격 | 연 $5,000~50,000 |
| 타겟 | 중견기업, 스케일업 |
| 난이도 | 중간 |

판매 가치: 조직의 기억 상실 방지. 시니어 퇴사 시
"이 코드가 왜 이렇게 되었는지 아무도 모른다"는 상황을 막고,
신규 입사자가 "왜 이렇게 되었나"를 검색으로 자답하게 한다.

번들: MemPalace + 커스텀 마이닝 파이프라인 + 온보딩 대시보드 + 교육.

#### 아이디어 9. 플러그인 / 백엔드 마켓플레이스

| 항목 | 내용 |
| --- | --- |
| 가격 | 거래 수수료 15~30% |
| 난이도 | 높음 (생태계 선행 필요) |

`backends/`와 `sources/`의 엔트리포인트 기반 유료 플러그인 생태계가 가능하지만
사용자 규모 확보가 선행 조건이다.

### 추천 실행 로드맵

| 시기 | 실행 | 근거 |
| --- | --- | --- |
| 0~1개월 | 아이디어 2 (교육 콘텐츠) | 리스크 0, 한국어 시장 선점, 시장 검증 동시 수행 |
| 1~3개월 | 아이디어 1 (컨설팅) 병행 | 콘텐츠가 리드를 자동 생성 → 전환 |
| 2~5개월 | 아이디어 4 (GUI 앱) | 가장 확실한 수요 + 커뮤니티 자산 + 상위 티어 진입 다리 |
| 4~8개월 | 아이디어 3 또는 6 | GUI 앱 피드백으로 방향 결정 |
| 8개월~ | 아이디어 5 (팀 호스팅) | 운영 부담이 크므로 고객 기반 확보 후 진입 |

### 핵심 전략 요약

> **엔진은 수정하지 않고, 보이는 것(GUI)과 아는 것(교육·현지화·도메인 지식)을 판다.**

---

## 11. 주의사항 및 리스크

### 기술적 주의사항

| 항목 | 내용 |
| --- | --- |
| MCP 연결 | 분석 세션에서 `mempalace-mcp` 실행파일이 PATH에 없어 MCP 서버 연결 실패. `uv tool install mempalace` 필요 |
| 포크 | 본 저장소는 포크본. 최신 업데이트는 upstream에서 받아야 함 |
| Beta 단계 | `Development Status :: 4 - Beta`. 원작자도 "중요 파일로 먼저 테스트하지 말라"고 명시 |
| Termux | ChromaDB/ONNX에 Android wheel 없음 → Debian PRoot 필요 |
| GPU 이미지 | x86_64 전용 (`onnxruntime-gpu` aarch64 wheel 없음) |
| Docker 마운트 | 리눅스에서 uid 1000 읽기 권한 필요. `--user` 우회 금지 |
| writer lease | 하나의 팰리스에 두 개의 `serve` 프로세스를 붙이면 안 됨 |
| 임베딩 전환 | 모델 변경 시 벡터 공간이 달라져 `mempalace repair rebuild-index` 필요 |
| 디스크 | 임베딩 모델 약 300MB + 팰리스 데이터 |
| 사칭 사이트 | `mempalace.tech` 등은 사칭. 공식은 GitHub / PyPI / mempalaceofficial.com |

### 사업 리스크

| 리스크 | 대응 |
| --- | --- |
| upstream이 유사 기능을 공식 출시 | 커뮤니티에 사전 공유·협력, 틈새(한국어·도메인)로 방어 |
| MIT라 모방 용이 | 진입장벽 구축 — 한국어 처리, 도메인 지식, 브랜드 |
| 철학 위반 시 커뮤니티 반발 | E2E 암호화, 요약 금지, 텔레메트리 금지 준수 |
| Beta 단계 API 불안정 | 버전 핀 고정, 어댑터 레이어로 격리 |
| 원 프로젝트와의 관계 | 포크 사실 명시, 상표 사용 주의, PR 기여로 선순환 |
| 기술 리스크 | 엔진 재작성 회피로 최소화 |

---

## 12. 참고 링크

### GitHub

| 항목 | URL |
| --- | --- |
| **본 저장소 (포크)** | <https://github.com/bmshin94/mempalace> |
| **원본 저장소 (upstream)** | <https://github.com/MemPalace/mempalace> |
| 릴리즈 | <https://github.com/MemPalace/mempalace/releases> |
| 이슈 트래커 | <https://github.com/MemPalace/mempalace/issues> |
| Claude Code 세션 만료 Discussion | <https://github.com/MemPalace/mempalace/discussions/1388> |

### 공식 배포 채널 (이 세 곳만 공식)

| 항목 | URL |
| --- | --- |
| GitHub | <https://github.com/MemPalace/mempalace> |
| PyPI | <https://pypi.org/project/mempalace/> |
| 공식 문서 | <https://mempalaceofficial.com> |
| Docker 이미지 | `ghcr.io/mempalace/mempalace:latest` |
| Discord | <https://discord.com/invite/ycTQQCu6kn> |

### 주요 문서

| 주제 | URL |
| --- | --- |
| 시작하기 | <https://mempalaceofficial.com/guide/getting-started.html> |
| Claude Code 보존 체크리스트 | <https://mempalaceofficial.com/guide/claude-code-retention.html> |
| 훅 설정 | <https://mempalaceofficial.com/guide/hooks.html> |
| Cursor 훅 | <https://mempalaceofficial.com/guide/cursor-hooks.html> |
| Antigravity | <https://mempalaceofficial.com/guide/antigravity.html> |
| CLI 레퍼런스 | <https://mempalaceofficial.com/reference/cli.html> |
| Python API | <https://mempalaceofficial.com/reference/python-api.html> |
| MCP 툴 목록 | <https://mempalaceofficial.com/reference/mcp-tools.html> |
| 팰리스 개념 | <https://mempalaceofficial.com/concepts/the-palace.html> |
| 지식그래프 | <https://mempalaceofficial.com/concepts/knowledge-graph.html> |
| 에이전트 | <https://mempalaceofficial.com/concepts/agents.html> |

### 저장소 내 문서

| 주제 | 경로 |
| --- | --- |
| 벤치마크 방법론 | `benchmarks/BENCHMARKS.md` |
| 공개 정정·철회 기록 | `docs/HISTORY.md` |
| 릴리즈 노트 | `CHANGELOG.md` |
| 설계 원칙 | `CLAUDE.md` |
| 설계 배경 | `MISSION.md` |
| 기여 가이드 | `CONTRIBUTING.md` |
| 보안 정책 | `SECURITY.md` |
| RFC | `docs/rfcs/001~005` |
| 네이티브 엔진 | `crates/README.md` |
| Termux 설치 | `website/guide/termux.md` |

---

## 부록 — 저장소 식별 정보

| 항목 | 값 |
| --- | --- |
| 분석 대상 | `bmshin94/mempalace` (<https://github.com/bmshin94/mempalace>) |
| 원본 | `MemPalace/mempalace` (<https://github.com/MemPalace/mempalace>) |
| 버전 | 3.10.0 |
| 라이선스 | MIT |
| 기본 브랜치 | `develop` |
| 분석 브랜치 | `claude/inspiring-ptolemy-sb5bsc` |
| 작성일 | 2026-10-07 |
