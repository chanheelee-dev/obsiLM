# obsiLM Specification

> obsidian-based notebookLM — 하나의 주제를 하나의 Obsidian Vault로 관리하는 로컬 AI 노트북

---

## 1. Overview

### 핵심 개념

**1 Topic = 1 Notebook = 1 Obsidian Vault**

Google NotebookLM에서 영감을 받아, 사용자가 특정 주제에 관한 소스(PDF, URL, 유튜브 등)를 모아두면 AI가 해당 소스들을 기반으로 질문 응답, 요약, 학습 자료 생성을 수행한다. 모든 데이터는 Obsidian Vault 구조의 로컬 마크다운 파일로 저장되어 Obsidian GUI로도 열람·편집 가능하다.

### 목표

- 로컬 우선(Local-first): 모든 노트와 소스는 사용자 파일시스템에 저장
- Obsidian 생태계 활용: Obsidian 공식 CLI(v1.12.0+, 2026년 2월 출시) 및 vault 구조 사용
- AI 기반 지식 정리: 소스 기반 Q&A, 요약, 학습 자료 자동 생성

### 기술 스택

| 구성요소 | 선택 |
|---|---|
| 언어 | Python 3.12+ |
| CLI 프레임워크 | [Typer](https://typer.tiangolo.com/) |
| Obsidian 연동 | Obsidian 공식 CLI (`obsidian` command) |
| AI API | Claude API (기본) |
| 의존성 관리 | `uv` |

---

## 2. 핵심 개념 상세

### Notebook (= Obsidian Vault)

각 노트북은 독립적인 Obsidian vault 폴더로 표현된다.

**기본 저장 경로**: `~/obsiLM/notebooks/<notebook-slug>/`

**Vault 내부 구조**:

```
<notebook-slug>/
├── .obsidian/              # Obsidian 설정 (자동 생성)
│   └── app.json
├── .obsilm                 # obsiLM vault 식별 flag 파일 (YAML)
├── sources/                # 추가된 소스들 (마크다운으로 변환)
│   ├── source-001.md
│   ├── source-002.md
│   └── ...
├── generated/              # AI가 생성한 콘텐츠
│   ├── summary.md
│   ├── study-guide.md
│   └── flashcards.md
├── memory/                 # AI 기억 저장소
│   ├── 2026-03-18.md
│   └── ...
├── chat/                   # 대화 기록
│   ├── 2026-03-18.md
│   └── ...
└── notebook.md             # 노트북 메타데이터 (제목, 생성일, 태그, 소스 목록)
```

### `.obsilm` flag 파일

루트에 위치하는 YAML 파일로, 해당 폴더가 obsiLM vault임을 식별한다.
일반 Obsidian vault와 구분하는 기준이 되며, `obsilm list` 등의 명령이 이 파일의 존재 여부로 노트북을 탐색한다.

```yaml
version: "1.0"
slug: ai-safety-research
title: "AI Safety Research"
created: 2026-03-18T09:00:00
model: claude-opus-4-6
```

### `memory/` 폴더

AI가 대화 컨텍스트를 넘나들며 활용할 정보를 날짜별 마크다운 파일로 저장한다.

- 사용자가 채팅 중 `/remember` 명령으로 명시적으로 기억시킨 내용
- 소스 인덱스 캐시 (재처리 방지용 요약)
- 향후: AI가 자동으로 중요 인사이트를 누적하는 장기 기억

```markdown
---
date: 2026-03-18
type: memory
---

# 2026-03-18 기억

- RLHF는 source-001, source-002 양쪽에서 핵심 기법으로 언급됨
- 사용자가 특히 scalable oversight 부분에 관심을 보임
```

### Source 노트 포맷

모든 소스는 YAML frontmatter를 가진 마크다운 파일로 저장된다.

```markdown
---
id: source-001
type: url          # url | pdf | youtube | file
origin: https://example.com/article
title: "예시 아티클 제목"
added: 2026-03-18T10:00:00
tags: [ai, research]
---

# 예시 아티클 제목

(소스에서 추출한 텍스트 본문)
```

### notebook.md 포맷

```markdown
---
title: "AI Safety Research"
slug: ai-safety-research
created: 2026-03-18T09:00:00
model: claude-opus-4-6
tags: [ai, safety, research]
source_count: 3
---

# AI Safety Research

(노트북 설명)
```

---

## 3. CLI Interface

### 설계 원칙: activate 방식

매번 notebook ID를 입력하는 대신, 먼저 노트북을 **activate**해두면 이후 모든 명령이 활성 노트북에 자동으로 적용된다.
활성 노트북 상태는 `~/.obsilm/state.toml`에 저장된다.

```toml
[active]
notebook = "ai-safety-research"
path = "/Users/user/obsiLM/notebooks/ai-safety-research"
```

### 명령어 구조

```
obsilm <command> [options]
```

---

### 3.1 Notebook 관리

#### `obsilm create <title>`
새 노트북(vault)을 생성하고 자동으로 activate한다.

```bash
$ obsilm create "AI Safety Research"
✓ Created and activated: ai-safety-research
  Path: ~/obsiLM/notebooks/ai-safety-research/
```

- `<title>`에서 slug를 자동 생성 (소문자, 공백→하이픈)
- `.obsidian/`, `.obsilm`, `notebook.md`, `sources/`, `generated/`, `memory/`, `chat/` 자동 생성

#### `obsilm activate <notebook>`
기존 노트북을 활성화한다.

```bash
$ obsilm activate llm-architecture
✓ Activated: llm-architecture
```

#### `obsilm deactivate`
활성 노트북을 해제한다.

```bash
$ obsilm deactivate
✓ Deactivated (was: ai-safety-research)
```

#### `obsilm status`
현재 활성 노트북과 기본 정보를 출력한다.

```bash
$ obsilm status
Active notebook: ai-safety-research
  Path: ~/obsiLM/notebooks/ai-safety-research
  Sources: 3 | Generated: 2 | Model: claude-opus-4-6
```

활성 노트북이 없으면:
```bash
$ obsilm status
No active notebook. Use 'obsilm activate <notebook>' or 'obsilm create <title>'.
```

#### `obsilm list`
`.obsilm` 파일이 있는 모든 노트북 목록을 표시한다.

```bash
$ obsilm list
  NOTEBOOK                  SOURCES  CREATED
* ai-safety-research        5        2026-03-18   ← 활성 노트북 (*로 표시)
  llm-architecture          3        2026-03-17
```

#### `obsilm delete [notebook]`
노트북을 삭제한다. `notebook` 생략 시 활성 노트북 대상. 확인 프롬프트 필수.

```bash
$ obsilm delete
Delete active notebook 'ai-safety-research'? This cannot be undone. [y/N]: y
✓ Deleted: ai-safety-research
```

#### `obsilm open [notebook]`
Obsidian GUI로 vault를 연다. `notebook` 생략 시 활성 노트북 대상.

```bash
$ obsilm open
# Obsidian GUI가 ai-safety-research vault를 열음
```

#### `obsilm info [notebook]`
노트북 상세 정보를 출력한다. `notebook` 생략 시 활성 노트북 대상.

---

### 3.2 Source 관리

아래 명령들은 모두 **활성 노트북**에 적용된다. `--notebook <slug>` 옵션으로 명시적 대상 지정도 가능하다.

#### `obsilm add --url <url>`

```bash
$ obsilm add --url https://arxiv.org/abs/2401.xxxxx
⠹ Fetching URL...
⠹ Converting to markdown...
✓ Added source to [ai-safety-research]: source-003.md (title: "Scalable Oversight...")
```

#### `obsilm add --file <path>`

```bash
$ obsilm add --file ./paper.pdf
⠹ Extracting text from PDF...
✓ Added source to [ai-safety-research]: source-004.md (title: "paper.pdf")
```

지원 파일 형식: `.pdf`, `.txt`, `.md`

#### `obsilm add --youtube <url>`

```bash
$ obsilm add --youtube https://youtube.com/watch?v=xxxxx
⠹ Fetching transcript...
✓ Added source to [ai-safety-research]: source-005.md (title: "Video Title")
```

#### `obsilm sources`
활성 노트북의 소스 목록을 출력한다.

```bash
$ obsilm sources
Notebook: ai-safety-research

ID           TYPE     TITLE                          ADDED
source-001   url      Scalable Oversight Paper       2026-03-18
source-002   pdf      alignment_survey.pdf           2026-03-18
source-003   youtube  AI Safety Lecture #1           2026-03-18
```

#### `obsilm remove-source <source-id>`
활성 노트북에서 소스를 제거한다.

---

### 3.3 AI Chat

#### `obsilm chat`
활성 노트북의 소스 기반 대화형 CLI 채팅 세션을 시작한다.

```bash
$ obsilm chat
obsiLM Chat — ai-safety-research (3 sources loaded)
Type 'exit' to quit, '/save' to save, '/remember <text>' to memorize.

You: What are the main arguments across these sources?

AI: Based on the sources you've added, here are the main arguments:
    1. [source-001] ...
    2. [source-002] ...
    (소스 인용 포함)

You: /remember RLHF가 source-001, 002 모두에서 핵심 기법으로 언급됨
✓ Memorized → memory/2026-03-18.md

You: /save
✓ Conversation saved: chat/2026-03-18.md

You: exit
```

- 모든 소스(`sources/*.md`)와 `memory/` 내용을 컨텍스트로 사용
- 응답에 `[source-001]` 형태의 인용 포함
- `/remember <text>`: 해당 내용을 `memory/YYYY-MM-DD.md`에 기록
- `/save`: 현재까지 대화를 `chat/YYYY-MM-DD.md`에 저장

---

### 3.4 콘텐츠 생성

아래 명령들은 모두 **활성 노트북**에 적용된다.

#### `obsilm generate summary`

```bash
$ obsilm generate summary
⠹ Generating summary...
✓ Saved: generated/summary.md
```

#### `obsilm generate study-guide`

```bash
$ obsilm generate study-guide
⠹ Generating study guide...
✓ Saved: generated/study-guide.md
```

#### `obsilm generate flashcards`

```bash
$ obsilm generate flashcards
⠹ Generating flashcards...
✓ Saved: generated/flashcards.md (20 cards)
```

생성된 파일 포맷 예시 (`flashcards.md`):
```markdown
# Flashcards

## Q: RLHF란 무엇인가?
A: Reinforcement Learning from Human Feedback. 인간의 피드백을 강화학습에 활용하는 기법.
[source-001]

## Q: ...
A: ...
```

---

## 4. Technical Architecture

### 시스템 구성

```
┌─────────────┐     ┌──────────────────┐     ┌─────────────────┐
│  obsilm CLI │────▶│  Core Engine     │────▶│  Obsidian Vault │
│  (Typer)    │     │  - SourceManager │     │  (Local Files)  │
└─────────────┘     │  - ChatEngine    │     └─────────────────┘
                    │  - MemoryManager │
                    │  - Generator     │
                    └────────┬─────────┘
                             │
                    ┌────────▼─────────┐
                    │   Claude API     │
                    │  (Anthropic)     │
                    └──────────────────┘
```

### 모듈 구조

```
obsilm/
├── cli.py              # Typer CLI 진입점
├── config.py           # 설정 관리 (~/.obsilm/config.toml, state.toml)
├── notebook.py         # Notebook(Vault) CRUD + activate/deactivate
├── source/
│   ├── manager.py      # 소스 추가/삭제/목록
│   ├── fetcher.py      # URL 크롤링, PDF 추출, YouTube 자막
│   └── converter.py    # 각 소스 타입 → 마크다운 변환
├── ai/
│   ├── client.py       # Claude API 클라이언트
│   ├── chat.py         # 대화 세션 관리
│   ├── memory.py       # memory/ 폴더 읽기/쓰기
│   └── generator.py    # 콘텐츠 생성 (summary, study-guide, flashcards)
└── obsidian.py         # Obsidian CLI 래퍼
```

### 설정 파일 구조

```
~/.obsilm/
├── config.toml   # API 키, 노트북 루트 경로, 기본 모델
└── state.toml    # 현재 활성 노트북
```

**`~/.obsilm/config.toml`**:

```toml
[general]
notebooks_dir = "~/obsiLM/notebooks"
default_model = "claude-opus-4-6"

[api]
anthropic_api_key = ""  # 또는 환경변수 ANTHROPIC_API_KEY
```

**`~/.obsilm/state.toml`**:

```toml
[active]
notebook = "ai-safety-research"
path = "/Users/user/obsiLM/notebooks/ai-safety-research"
```

### Obsidian CLI 연동

Obsidian 공식 CLI(v1.12.0+)를 사용한다.

```python
# obsidian.py
import subprocess

def open_vault(vault_path: str):
    """Obsidian GUI로 vault 열기"""
    subprocess.run(["obsidian", "open", vault_path])
```

노트 파일 생성/수정은 직접 파일 I/O로 처리한다 (마크다운 파일 직접 조작).
Obsidian CLI의 고급 기능(JS 실행, 플러그인 제어)은 향후 버전에서 활용한다.

---

## 5. User Flow 시나리오

### 시나리오: AI Safety 연구 정리

```bash
# 1. 노트북 생성 (자동 activate)
$ obsilm create "AI Safety Research"
✓ Created and activated: ai-safety-research

# 2. 현재 상태 확인
$ obsilm status
Active notebook: ai-safety-research
  Sources: 0 | Model: claude-opus-4-6

# 3. 소스 추가 (notebook ID 불필요)
$ obsilm add --url https://arxiv.org/abs/2401.xxxxx
✓ Added source to [ai-safety-research]: source-001.md

$ obsilm add --file alignment_survey.pdf
✓ Added source to [ai-safety-research]: source-002.md

$ obsilm add --youtube https://youtube.com/watch?v=xxxxx
✓ Added source to [ai-safety-research]: source-003.md

# 4. AI와 대화
$ obsilm chat
obsiLM Chat — ai-safety-research (3 sources loaded)

You: 이 소스들의 핵심 주장을 3줄로 요약해줘
AI: [source-001, source-002 인용] ...

You: /remember RLHF가 세 소스 모두에서 핵심 기법으로 언급됨
✓ Memorized → memory/2026-03-18.md

You: exit

# 5. 학습 자료 생성
$ obsilm generate study-guide
✓ Saved: generated/study-guide.md

$ obsilm generate flashcards
✓ Saved: generated/flashcards.md (15 cards)

# 6. 다른 노트북으로 전환
$ obsilm activate llm-architecture
✓ Activated: llm-architecture

$ obsilm add --youtube https://youtube.com/watch?v=yyyyy
✓ Added source to [llm-architecture]: source-001.md

# 7. Obsidian으로 열기
$ obsilm open
# Obsidian GUI가 llm-architecture vault를 열음
```

---

## 6. MVP 범위 및 우선순위

### Phase 1 — MVP

| 기능 | 우선순위 |
|---|---|
| 노트북 생성/목록/삭제 | P0 |
| activate / deactivate / status | P0 |
| URL 소스 추가 (크롤링) | P0 |
| PDF 소스 추가 | P0 |
| AI Chat (소스 기반 Q&A) | P0 |
| 요약 생성 (`generate summary`) | P0 |
| memory/ 기본 구현 (`/remember`) | P1 |
| Obsidian으로 열기 | P1 |
| YouTube 소스 추가 | P1 |
| Study Guide 생성 | P1 |
| Flashcard 생성 | P1 |

### Phase 2 — 확장

| 기능 | 비고 |
|---|---|
| 오디오 Overview (팟캐스트) | NotebookLM의 대표 기능 |
| 마인드맵 생성 | Obsidian Canvas 활용 |
| Obsidian 플러그인 자동 설치 | Obsidian CLI JS 실행 기능 |
| 컨텍스트 윈도우 초과 처리 | 청킹 + 벡터 검색 (RAG) |
| memory/ 고도화 | AI 자동 장기 기억 누적 |
| Web UI | (선택 사항) |

---

## 7. 비기능 요구사항

- **로컬 우선**: 인터넷 연결 없이도 기존 소스/생성물 열람 가능 (AI 기능 제외)
- **Obsidian 호환**: 생성된 모든 파일은 Obsidian에서 그대로 열람 가능한 표준 마크다운
- **확장성**: 새 소스 타입, 새 생성 타입을 플러그인 방식으로 추가 가능한 구조
- **API 키 보안**: API 키는 환경변수 또는 설정 파일에 저장, 코드에 하드코딩 금지
- **vault 식별**: `.obsilm` 파일이 있는 폴더만 obsiLM notebook으로 인식, 일반 Obsidian vault와 충돌 없음
