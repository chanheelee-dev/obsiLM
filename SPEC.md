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
├── sources/                # 추가된 소스들 (마크다운으로 변환)
│   ├── source-001.md
│   ├── source-002.md
│   └── ...
├── generated/              # AI가 생성한 콘텐츠
│   ├── summary.md
│   ├── study-guide.md
│   └── flashcards.md
├── chat/                   # 대화 기록
│   ├── 2026-03-18.md
│   └── ...
└── notebook.md             # 노트북 메타데이터 (제목, 생성일, 태그, 소스 목록)
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

### 명령어 구조

```
obsilm <command> [options]
```

### 3.1 Notebook 관리

#### `obsilm create <title>`
새 노트북(vault)을 생성한다.

```bash
$ obsilm create "AI Safety Research"
✓ Created notebook: ai-safety-research
  Path: ~/obsiLM/notebooks/ai-safety-research/
```

- `<title>`에서 slug를 자동 생성 (소문자, 공백→하이픈)
- `.obsidian/` 폴더와 `notebook.md` 자동 생성

#### `obsilm list`
모든 노트북 목록을 표시한다.

```bash
$ obsilm list
NOTEBOOK                  SOURCES  CREATED
ai-safety-research        5        2026-03-18
llm-architecture          3        2026-03-17
```

#### `obsilm delete <notebook>`
노트북을 삭제한다. 확인 프롬프트 필수.

#### `obsilm open <notebook>`
Obsidian GUI로 해당 vault를 연다.

```bash
$ obsilm open ai-safety-research
# Obsidian CLI 호출: obsidian open ~/obsiLM/notebooks/ai-safety-research
```

#### `obsilm info <notebook>`
노트북 상세 정보(소스 목록, 생성일, 모델 등)를 출력한다.

---

### 3.2 Source 관리

#### `obsilm add <notebook> --url <url>`
웹 URL을 크롤링해 마크다운으로 변환 후 소스로 추가한다.

```bash
$ obsilm add ai-safety-research --url https://arxiv.org/abs/2401.xxxxx
⠹ Fetching URL...
⠹ Converting to markdown...
✓ Added source: source-003.md (title: "Scalable Oversight...")
```

#### `obsilm add <notebook> --file <path>`
로컬 파일(PDF, 텍스트, 마크다운)을 소스로 추가한다.

```bash
$ obsilm add ai-safety-research --file ./paper.pdf
⠹ Extracting text from PDF...
✓ Added source: source-004.md (title: "paper.pdf")
```

지원 파일 형식: `.pdf`, `.txt`, `.md`

#### `obsilm add <notebook> --youtube <url>`
YouTube URL에서 자막(transcript)을 추출해 소스로 추가한다.

```bash
$ obsilm add ai-safety-research --youtube https://youtube.com/watch?v=xxxxx
⠹ Fetching transcript...
✓ Added source: source-005.md (title: "Video Title")
```

#### `obsilm sources <notebook>`
노트북의 소스 목록을 출력한다.

```bash
$ obsilm sources ai-safety-research
ID           TYPE     TITLE                          ADDED
source-001   url      Scalable Oversight Paper       2026-03-18
source-002   pdf      alignment_survey.pdf           2026-03-18
source-003   youtube  AI Safety Lecture #1           2026-03-18
```

#### `obsilm remove-source <notebook> <source-id>`
소스를 노트북에서 제거한다.

---

### 3.3 AI Chat

#### `obsilm chat <notebook>`
소스 기반 대화형 CLI 채팅 세션을 시작한다.

```bash
$ obsilm chat ai-safety-research
obsiLM Chat — ai-safety-research (3 sources loaded)
Type 'exit' to quit, '/save' to save conversation.

You: What are the main arguments across these sources?

AI: Based on the sources you've added, here are the main arguments:
    1. [source-001] ...
    2. [source-002] ...
    (소스 인용 포함)

You: /save
✓ Conversation saved: chat/2026-03-18.md

You: exit
```

- 모든 소스(`sources/*.md`)를 컨텍스트로 사용
- 응답에 `[source-001]` 형태의 인용 포함
- 대화 내용은 세션 종료 시 또는 `/save` 명령 시 `chat/YYYY-MM-DD.md`에 저장

---

### 3.4 콘텐츠 생성

#### `obsilm generate summary <notebook>`
모든 소스를 종합한 요약문을 생성한다.

```bash
$ obsilm generate summary ai-safety-research
⠹ Generating summary...
✓ Saved: generated/summary.md
```

#### `obsilm generate study-guide <notebook>`
핵심 개념, 용어 정리, 주요 논점을 담은 학습 가이드를 생성한다.

```bash
$ obsilm generate study-guide ai-safety-research
⠹ Generating study guide...
✓ Saved: generated/study-guide.md
```

#### `obsilm generate flashcards <notebook>`
Q&A 형식의 플래시카드를 생성한다. Obsidian의 플래시카드 플러그인 호환 형식.

```bash
$ obsilm generate flashcards ai-safety-research
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
├── config.py           # 설정 관리 (~/.obsilm/config.toml)
├── notebook.py         # Notebook(Vault) CRUD
├── source/
│   ├── manager.py      # 소스 추가/삭제/목록
│   ├── fetcher.py      # URL 크롤링, PDF 추출, YouTube 자막
│   └── converter.py    # 각 소스 타입 → 마크다운 변환
├── ai/
│   ├── client.py       # Claude API 클라이언트
│   ├── chat.py         # 대화 세션 관리
│   └── generator.py    # 콘텐츠 생성 (summary, study-guide, flashcards)
└── obsidian.py         # Obsidian CLI 래퍼
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

### 설정 파일

**`~/.obsilm/config.toml`**:

```toml
[general]
notebooks_dir = "~/obsiLM/notebooks"
default_model = "claude-opus-4-6"

[api]
anthropic_api_key = ""  # 또는 환경변수 ANTHROPIC_API_KEY
```

---

## 5. User Flow 시나리오

### 시나리오: AI Safety 연구 정리

```bash
# 1. 노트북 생성
$ obsilm create "AI Safety Research"
✓ Created notebook: ai-safety-research

# 2. 소스 추가
$ obsilm add ai-safety-research --url https://arxiv.org/abs/2401.xxxxx
✓ Added source: source-001.md

$ obsilm add ai-safety-research --file alignment_survey.pdf
✓ Added source: source-002.md

$ obsilm add ai-safety-research --youtube https://youtube.com/watch?v=xxxxx
✓ Added source: source-003.md

# 3. AI와 대화
$ obsilm chat ai-safety-research
You: 이 소스들의 핵심 주장을 3줄로 요약해줘
AI: [source-001, source-002 인용] ...

# 4. 학습 자료 생성
$ obsilm generate study-guide ai-safety-research
✓ Saved: generated/study-guide.md

$ obsilm generate flashcards ai-safety-research
✓ Saved: generated/flashcards.md (15 cards)

# 5. Obsidian으로 열어서 확인
$ obsilm open ai-safety-research
# Obsidian GUI가 해당 vault를 열음
```

---

## 6. MVP 범위 및 우선순위

### Phase 1 — MVP

| 기능 | 우선순위 |
|---|---|
| 노트북 생성/목록/삭제 | P0 |
| URL 소스 추가 (크롤링) | P0 |
| PDF 소스 추가 | P0 |
| AI Chat (소스 기반 Q&A) | P0 |
| 요약 생성 (`generate summary`) | P0 |
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
| Web UI | (선택 사항) |

---

## 7. 비기능 요구사항

- **로컬 우선**: 인터넷 연결 없이도 기존 소스/생성물 열람 가능 (AI 기능 제외)
- **Obsidian 호환**: 생성된 모든 파일은 Obsidian에서 그대로 열람 가능한 표준 마크다운
- **확장성**: 새 소스 타입, 새 생성 타입을 플러그인 방식으로 추가 가능한 구조
- **API 키 보안**: API 키는 환경변수 또는 설정 파일에 저장, 코드에 하드코딩 금지
