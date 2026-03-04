# workspace-rag: Skills 기반 경량 RAG 시스템
  
[출처](https://zenn.dev/karaage0703/articles/d7eaf62437185d )  
  
---

## 개요
가볍고 간편한 방식을 모색한 결과로 탄생한 것이 **workspace-rag**다. PostgreSQL이나 Docker 없이 `uv`와 SQLite만으로 동작한다.   
https://github.com/karaage0703/ai-assistant-workspace  

---

## 아키텍처 및 기술 스택
RAG 구성 자체는 MCP 버전과 동일하지만, 인터페이스를 MCP 대신 **Skills** 방식으로 전환한 것이 핵심이다. 기술 스택은 다음과 같다.

- DB: SQLite (파일 1개)
- 임베딩 모델: `intfloat/multilingual-e5-small` (384차원, 약 500MB)
- 검색: 코사인 유사도 기반 벡터 검색

---

## MCP RAG 서버와의 비교

| 항목 | MCP RAG 서버 (이전) | workspace-rag (이번) |
|------|------|------|
| DB | PostgreSQL + pgvector (Docker 필수) | SQLite (파일 1개) |
| 모델 | multilingual-e5-large (1024차원, ~1.2GB) | multilingual-e5-small (384차원, ~500MB) |
| 인터페이스 | MCP 프로토콜 | AI 에이전트 Skills (CLI 명령어) |
| 셋업 | Docker, .env 설정, PostgreSQL 초기화 | `uv sync` 만으로 완료 |

가장 큰 차이는 **인터페이스**다. MCP 버전은 서버로서 상주 프로세스가 필요하지만, workspace-rag는 Skills로서 필요할 때만 커맨드를 실행하면 된다.

---

## Skills란
Skills는 Anthropic이 제창한 AI 에이전트 규격으로, AI 에이전트가 다양한 도구를 사용하기 위한 구조다. `SKILL.md`(사용 설명서)와 스크립트를 지정 디렉토리에 배치하는 것만으로 AI 에이전트가 자율적으로 활용할 수 있게 된다. 심볼릭 링크를 걸어두면 모든 에이전트에서 공통으로 사용할 수 있어 편리하다.  
  
AI 에이전트는 워크스페이스를 검색하고 싶을 때 아래 커맨드를 실행한다.  

```bash
uv run python workspace_rag.py search -w /path/to/workspace -q "작년 이벤트 발표 자료"
```

---

## 관련도 스코어 부여 검색 (R²AG)
검색 결과에 관련도 스코어를 부여하는 구조를 구현했다. 이는 EMNLP 2024에서 발표된 R²AG 아이디어를 간이 구현한 것이다.  

```
문서1 [관련도: 0.92 (高)]: 메모리 파일 관리 방법에 대해...
문서2 [관련도: 0.31 (低)]: 고양이 사진 찍는 팁...
질문: 메모리 파일 작성 방법은?
```

이 정보를 통해 AI 에이전트가 어떤 문서를 우선할지 판단하기 쉬워진다.

---

## 동작 확인
인덱스 생성은 최초 1회만 수 분 소요되며, 이후에는 차분(diff)만 처리하므로 수 초 만에 완료된다.

```bash
# 인덱스 생성
$ uv run python workspace_rag.py index -w ~/workspace
📊 인덱스 통계:
  신규: 342파일 / 업데이트: 0파일 / 스킵: 0파일
  총 청크: 2,847

# 검색
$ uv run python workspace_rag.py search -w ~/workspace -q "컨텍스트 엔지니어링" --r2ag
[1] [관련도: 0.94 (高)] skills/context-engineering/SKILL.md
[2] [관련도: 0.89 (高)] notes/20260115_context_tips.md
[3] [관련도: 0.72 (中)] memory/20260120.md
```


### 동작 확인 상세 설명

#### 인덱싱(Indexing)이란
RAG 시스템에서 검색을 하려면 먼저 문서들을 **벡터로 변환해서 DB에 저장**하는 사전 작업이 필요하다. 이 과정을 인덱싱이라고 한다. 도서관에 비유하면, 책을 꽂기 전에 **목록 카드를 만들어두는 작업**과 같다. 한 번 만들어두면 나중에 빠르게 찾을 수 있다.
  
#### 인덱스 생성 커맨드 분석

```bash
uv run python workspace_rag.py index -w ~/workspace
```

각 옵션의 의미는 다음과 같다.

- `uv run` — Python 패키지 매니저 `uv`를 통해 실행 (가상환경 자동 처리)
- `python workspace_rag.py` — workspace-rag의 메인 스크립트 실행
- `index` — "인덱스를 만들어라"는 서브 커맨드
- `-w ~/workspace` — 인덱싱할 대상 디렉토리 지정 (홈 디렉토리 아래의 `workspace` 폴더)

실행 결과로 출력되는 통계의 의미는 다음과 같다.

- **신규 342파일** — 처음 발견되어 새로 인덱싱된 파일 수
- **업데이트 0파일** — 이전 인덱싱 이후 내용이 변경된 파일 수 (첫 실행이라 0)
- **스킵 0파일** — 변경 없어서 건너뛴 파일 수 (첫 실행이라 0)
- **총 청크 2,847** — 파일들을 잘게 쪼갠 조각(chunk)의 총 개수

여기서 **청크(chunk)** 란, 하나의 문서를 임베딩 모델이 처리하기 적합한 크기로 분할한 텍스트 조각을 말한다. 예를 들어 긴 마크다운 파일 1개가 여러 개의 청크로 나뉘어 저장된다. 342개 파일이 2,847개 청크로 분할되었으므로, 파일 1개당 평균 약 8개의 청크가 생성된 셈이다.

**차분(diff) 처리**란, 2회차 이후 실행 시 이전에 인덱싱한 파일의 해시값 등을 비교해서 **변경된 파일만 다시 인덱싱**하는 방식이다. 모든 파일을 처음부터 다시 처리하지 않기 때문에 2회차부터는 수 초 만에 완료된다.

  
#### 검색 커맨드 분석

```bash
uv run python workspace_rag.py search -w ~/workspace -q "컨텍스트 엔지니어링" --r2ag
```

각 옵션의 의미는 다음과 같다.

- `search` — "검색하라"는 서브 커맨드
- `-w ~/workspace` — 검색 대상 워크스페이스 경로
- `-q "컨텍스트 엔지니어링"` — 검색할 쿼리 문자열
- `--r2ag` — R²AG 방식으로 관련도 스코어를 함께 출력하는 옵션

검색 결과 해석은 다음과 같다.

```
[1] [관련도: 0.94 (高)] skills/context-engineering/SKILL.md
[2] [관련도: 0.89 (高)] notes/20260115_context_tips.md
[3] [관련도: 0.72 (中)] memory/20260120.md
```

- **관련도 수치(0~1)** — 쿼리 벡터와 문서 청크 벡터 간의 **코사인 유사도** 값이다. 1에 가까울수록 의미적으로 유사하다는 뜻이다.
- **高/中/低 레이블** — 수치를 사람(또는 AI)이 직관적으로 판단하기 쉽도록 구간별로 붙인 등급이다.
- **파일 경로** — 해당 청크가 속한 원본 파일의 경로를 보여준다.

결국 이 시스템이 하는 일은, 쿼리 문장을 벡터로 변환한 뒤 DB에 저장된 수천 개의 청크 벡터들과 유사도를 계산해서 **가장 관련성 높은 문서를 순위별로 반환**하는 것이다. AI 에이전트는 이 결과를 바탕으로 어떤 문서를 참고해서 답변할지 판단하게 된다.

---
  

## 상주 HTTP 서버를 통한 고속화
CLI 버전은 실행할 때마다 모델을 로드하기 때문에, 1회 검색에 수 초~10초 정도 소요된다. 실측 기준으로는 모델 로드에 약 8초, DB 조회 + 행렬 구성에 약 1초, 합계 약 9초였다(약 20만 청크 환경).

이를 해결하기 위해 `workspace_rag_server.py`라는 상주 HTTP 서버를 제공한다. 서버를 사용하면 검색이 약 0.1초로 대폭 단축된다. PC 시작 시 자동 실행되도록 systemd 유저 서비스로 등록하는 것도 가능하다.

---

## AI 에이전트에서의 활용
실제로는 AI 에이전트(예: Claude Code)가 Skills로서 커맨드를 자동 실행한다. "워크스페이스 RAG로 xxx를 찾아줘"라고 말하는 것만으로 AI가 관련 문서를 찾아 내용을 반영한 답변을 돌려준다. 인덱스 생성이나 HTTP 서버 상주화도 자연어로 지시하면 AI가 대신 실행해 준다.

---

## 참고 자료
- R²AG 논문: [R²AG: Incorporating Retrieval Information into RAG (EMNLP 2024)](https://arxiv.org/abs/2406.13249)
- workspace-rag GitHub 저장소 및 심볼릭 링크 설정 방법은 원문 리포지토리 참조    
  
  

## 📁 ai-assistant-workspace 정리
https://github.com/karaage0703/ai-assistant-workspace  
  
### **개요**
`ai-assistant-workspace`는 **AI 코딩 툴(Claude Code / Codex CLI / Gemini CLI)을 개인 비서처럼 활용하기 위한 스타터 키트**입니다. 처음 실행 시 대화형 인터페이스를 통해 사용자 맞춤형 AI 어시스턴트를 자동으로 구성해줍니다.

### **지원 AI 도구**
세 가지 AI CLI 도구를 지원하며, 각각에 대응하는 설정 파일이 존재합니다.

- **Claude Code** → `CLAUDE.md` (= `AGENTS.md` 심볼릭 링크)
- **Codex CLI** → `AGENTS.md`
- **Gemini CLI** → `GEMINI.md` (= `AGENTS.md` 심볼릭 링크)

  
### **디렉토리 구조**

```
ai-assistant-workspace/
├── AGENTS.md              # 공통 설정 파일
├── CLAUDE.md              # → AGENTS.md 심볼릭 링크
├── GEMINI.md              # → AGENTS.md 심볼릭 링크
├── BOOTSTRAP.md           # 초기 셋업 파일 (완료 후 삭제됨)
├── .claude/skills         # Claude Code 용 심볼릭 링크
├── .agents/skills         # Codex CLI 용 심볼릭 링크
├── .gemini/skills         # Gemini CLI 용 심볼릭 링크
├── memory/                # 일기·메모 저장 경로
├── notes/                 # 노트·리서치 저장 경로
└── skills/                # AI 확장 스킬 모음
```

---

### **주요 스킬 목록**
`skills/` 디렉토리 안에 다양한 기능 모듈이 포함되어 있습니다.

| 스킬 폴더 | 기능 설명 |
|---|---|
| `calendar/` | Google Calendar 등 ICS 형식의 일정 확인 |
| `diary/` | 일일 일기 작성 및 Notion 연동 관리 |
| `cat-diary/` | 고양이 사진 전송 시 자동 판별 후 Notion에 기록 |
| `note-taking/` | 조사 결과·아이디어·회의 메모 정리 및 저장 |
| `notion-manager/` | Notion 페이지 검색·생성·파일 업로드 |
| `transcriber/` | 음성 파일을 텍스트로 변환 |
| `podcast/` | 팟캐스트 다운로드 및 요약 (프리셋 방송 포함) |
| `youtube-notes/` | YouTube 영상 내용을 노트로 정리 |
| `marp-slides/` | 마크다운으로 프레젠테이션 슬라이드 생성 |
| `tech-news-curation/` | 최신 AI·기술 뉴스 수집 및 소개 |
| `arxiv/` | arXiv 논문 검색·트렌드 파악·상세 분석 |
| `code-reviewer/` | Multi-AI(Claude/Codex/Gemini)로 PR 체계적 리뷰 |
| `github-repo-analyzer/` | GitHub 저장소 구조 및 기술 스택 분석 |
| `workspace-rag/` | 파일을 벡터 검색으로 횡단 검색 |
| `health-advisor/` | 식사·운동 기록 및 건강 조언 |
| `spontaneous-talk/` | AI가 자발적으로 말 걸기 (확률 판단 + cron 지원) |
| `skill-creator/` | 나만의 커스텀 스킬 생성 |
| `xangi-settings/` | 채팅에서 AI 어시스턴트 설정 변경 (xangi 전용) |

  
### **빠른 시작 (Quick Start)**

```bash
# 1. 레포지토리 클론
git clone https://github.com/karaage0703/ai-assistant-workspace
cd ai-assistant-workspace

# 2. AI 도구 실행 (셋 중 하나 선택)
claude    # Claude Code
codex     # Codex CLI
gemini    # Gemini CLI

# 3. 대화형 초기 셋업 자동 시작
# AI가 사용자 정보를 수집해 맞춤형 어시스턴트를 구성합니다
```

  
### **초기 셋업 흐름 (BOOTSTRAP.md)**
처음 실행 시 `BOOTSTRAP.md`를 통해 다음 단계로 초기화가 진행됩니다.

1. **인사** — AI가 친근하게 사용자에게 인사
2. **사용자 정보 수집** — 이름(닉네임), 관심 분야, 활용 목적, 말투 선호도 등 질문
3. **어시스턴트 이름·성격 결정** — 사용자가 직접 AI 이름과 분위기를 선택
4. **`AGENTS.md` 자동 업데이트** — 수집된 정보를 설정 파일에 반영
5. **셋업 완료** — `BOOTSTRAP.md` 삭제 후 완료 메시지 출력

  
### **커스터마이징**
`AGENTS.md` 내 **"자분에 대해"** 섹션을 편집하면 AI의 말투와 성격을 변경할 수 있습니다. 또한 `skills/` 디렉토리에 새 폴더를 만들고 `SKILL.md` 파일을 작성하는 것만으로 **커스텀 스킬을 추가**할 수 있어 높은 확장성을 제공합니다.  