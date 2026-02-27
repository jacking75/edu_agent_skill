# Claude Code Skills 완전 가이드 — SKILL.md 설계부터 실전 구현까지
  
[출처](https://zenn.dev/acntechjp/articles/554765e5a0d0be )  
  
---

## **개요:**
Claude Code는 터미널에서 동작하는 AI 에이전트다. 코드 읽기/쓰기뿐 아니라 PowerShell, Python, OS 커맨드 실행도 가능하다. 그러나 프로젝트 고유의 업무 절차나 툴 연계 방법을 Claude가 처음부터 알고 있지는 않다. 이를 해결하는 것이 **Skills**다.

---

## **Skills란:**
Skills는 Claude Code에 "특정 업무를 어떻게 실행하는지"를 가르치는 구조다. `SKILL.md` 파일에 절차를 작성해두면, Claude가 해당 스킬을 자율적으로 선택해 PowerShell·Python 스크립트를 실행하며 업무를 수행한다. Anthropic 공식 블로그에서는 Skills를 **"신규 팀원을 위한 온보딩 가이드"**에 비유한다. 매뉴얼(`SKILL.md`)과 툴(스크립트)을 세트로 건네면, Claude가 스스로 판단해 작업을 진행한다.

---

## **MCP vs Skills:**
Claude Code의 확장 구조에는 **MCP(Model Context Protocol)**와 **Skills** 두 가지가 있다. MCP는 외부 툴과의 연계 프로토콜이고, Skills는 **"파일시스템 기반의 지식 패키지"**다. Skills의 본질은 Claude Code가 필요할 때 동적으로 읽어들이는 **"업무 매뉴얼 + 툴 일식"**이다.

---

## **핵심 설계 사상 — Progressive Disclosure (단계적 공개):**
Skills의 가장 중요한 설계 원칙이다. "필요한 정보를, 필요할 때, 필요한 만큼만 읽어들인다"는 원칙이다.

```
레벨 1: 프론트매터 (name + description)  → 항상 컨텍스트에 존재 (~100 토큰)
레벨 2: SKILL.md 본문                   → 스킬 기동 시 읽어들임
레벨 3: 보조 파일 (scripts/, references/) → 필요에 따라 읽어들이거나 실행
```

LLM에는 **컨텍스트 윈도우**라는 용량 제한이 있다. 모든 스킬 정보를 처음부터 읽어들이면 컨텍스트를 압박해 정확도가 떨어진다. Progressive Disclosure에 의해, 스킬을 아무리 많이 설치해도 **실제로 컨텍스트를 소비하는 건 사용되는 스킬만**이다. 또한 Python 스크립트 등의 실행 파일은 **컨텍스트에 읽어들이지 않고 실행**할 수 있어, 스크립트 코드 본체는 컨텍스트 윈도우를 소비하지 않고 실행 결과(출력)만 컨텍스트에 들어온다.

---

## **기동 방식 — 모델 기동 vs 유저 기동:**

| 기동 방식 | 설명 | 예시 |
|---|---|---|
| **모델 기동** | Claude가 대화 맥락에서 스킬을 자동 선택해 실행 | "SharePoint에서 파일 가져와" → Claude가 `download-sharepoint` 선택 |
| **유저 기동** | `/스킬명`으로 명시적으로 호출 | `/download-sharepoint` 입력 |

모델 기동의 판단 기준은 SKILL.md의 **프론트매터(`name` / `description`)**다. Claude Code는 기동 시 모든 스킬의 프론트매터를 읽어들여, 사용자 지시에 가장 적합한 스킬을 선택한다.

---

## **디렉터리 구성:**

```
.
├── .claude/
│   └── skills/                    # Claude Code 스킬 정의
│       ├── download-sharepoint/
│       │   └── SKILL.md
│       ├── excel-to-json/
│       │   └── SKILL.md
│       ├── member-lookup/
│       │   └── SKILL.md
│       └── search-location/
│           └── SKILL.md
├── scripts/                       # 실행 스크립트 (SKILL.md와 분리)
│   ├── 01_downloadExcelFromSharepoint.ps1
│   ├── 02_excel_to_json_python.py
│   ├── 03_member_lookup.py
│   └── 04_search_location.py
├── botwork/                       # 스크립트 입출력 디렉터리 (.gitignore 대상)
├── .env                           # 환경변수 (.gitignore 대상)
├── .env.example                   # 환경변수 템플릿
└── README.md
```

**중요 포인트:** `SKILL.md`(`.claude/skills/` 하위)와 스크립트(`scripts/` 하위)를 분리한다. `botwork/`는 스크립트 간 데이터 전달(Excel → JSON → 검색 결과)을 한 곳에서 관리하는 중간 저장소다. `.env`는 기밀 정보를 코드에서 분리하기 위한 것이다.

---

## **4가지 스킬 구현 예시:**

**① download-sharepoint — SharePoint에서 Excel 다운로드**

```markdown
---
name: download-sharepoint
description: SharePoint에서 Excel 파일을 다운로드 한다
disable-model-invocation: false
---

SharePoint에서 Excel 파일을 다운로드한다:

1. 아래의 PowerShell 스크립트를 실행한다:
   powershell -ExecutionPolicy Bypass -File "./scripts/01_downloadExcelFromSharepoint.ps1"
2. 다운로드 종료를 확인한다
3. 파일이 `botwork` 에 저장된 것을 보고한다
```

- `disable-model-invocation: false` → 모델 자동 선택 허용. "SharePoint에서 파일 가져와"라고 말해도 Claude가 이 스킬을 자동 선택한다
- `-ExecutionPolicy Bypass` 는 Windows 환경에서 스크립트 실행 정책을 일시 우회하는 옵션으로, Claude Code에서 실행 시 필수다

**② excel-to-json — Excel → JSON 변환**

```markdown
---
name: excel-to-json
description: 로컬의 Excel 파일 (MemberList）을 JSON 형식으로 변환한다.
disable-model-invocation: true
---

1. python "./scripts/02_excel_to_json_python.py" 를 실행한다
2. 変換完了を確認する
3. ファイルが `botwork` に保存されたことを報告する
```

- `disable-model-invocation: true` → **모델 자동 기동 비활성화**. 반드시 사용자가 `/excel-to-json`으로 명시적 호출해야 한다
- Excel → JSON 변환은 **중간 처리**이고, 잘못 실행되면 기존 JSON 파일을 덮어쓸 위험이 있기 때문에 의도적으로 자동 실행을 막는다
- 변환 로직은 모두 Python 스크립트에 위임하고, SKILL.md는 오케스트레이션(지휘)에만 집중한다
  
**③ member-lookup — ID로 멤버 정보 검색 (대화형)**

```markdown
---
name: member-lookup
description: ID를 지정하여 멤버 정보를 검색한다. 이름・팀・CL 등 각종 정보를 얻을 수 있다.  
---

## 手順
### STEP 1: IDを確認する
  検索したいメンバーのIDを入力してください。(例: D001)

### STEP 2: 取得フィールドを確認する
  1. 日本語名のみ（デフォルト）
  2. 基本情報（日本語名・拠点）

### STEP 3: コマンドを実行する
  1. python "./scripts/03_member_lookup.py" --id {ID}
  2. python "./scripts/03_member_lookup.py" --id {ID} --fields name_ja location

### STEP 4: 結果を伝える
  - status: ok → 정보를 정형화해서 전달
  - status: not_found → ID를 다시 확인하도록 안내
  - status: error → 에러 내용 전달
```

- STEP 형식으로 **대화 설계(UX 디자인)**까지 SKILL.md에 포함한다
- 사용자 선택에 따라 커맨드 인수가 달라지는 **조건 분기**를 기술한다
- Python 스크립트가 반환하는 JSON의 `status` 필드에 따른 응답 방법도 명시한다
- SKILL.md는 단순한 커맨드 실행 지시서가 아니라 **대화 시나리오 설계서**도 될 수 있다

**④ search-location — Playwright로 브라우저 조작**

```markdown
---
name: search-location
description: 都道府県名を指定してPlaywright（Chromium）でWikipediaを検索し、概要と目次を返す
---

### STEP 1: 都道府県を確認する
### STEP 2: python scripts/04_search_location.py --location "{都道府県}" を実行する
### STEP 3: 結果を伝える (status: ok / error)

## 初回セットアップ（必要な場合）
  pip install playwright
  python -m playwright install chromium
```

- 클라우드 AI 서비스에서 브라우저 직접 조작은 보안·기술적 이유로 어렵지만, Claude Code Skills는 **Windows 단말에서 로컬로 Playwright를 실행**하므로 브라우저 조작이 자연스럽게 가능하다
- 말미의 "초회 셋업" 섹션은 Claude에 대한 **폴백(fallback) 지시**다. Playwright 미설치 시 에러 발생에 대비해 설치 절차를 기술해두면, Claude가 에러 시 스스로 셋업을 제안할 수 있다

---

## **4가지 스킬 설계 패턴 비교:**

| 스킬 | 기동 방식 | 대화 유무 | 설계 의도 |
|---|---|---|---|
| download-sharepoint | 모델 기동 + 유저 기동 | 없음 | 단순 실행, 자동 선택 허용 |
| excel-to-json | 유저 기동만 | 없음 | 명시적 실행 강제 (부작용 방지) |
| member-lookup | 모델 기동 + 유저 기동 | 있음 (STEP 형식) | 대화 → 조건 분기 → 결과 핸들링 |
| search-location | 모델 기동 + 유저 기동 | 있음 (STEP 형식) | 브라우저 조작 + 에러 폴백 |

---

**스크립트 구현의 핵심 포인트:**

PowerShell 스크립트에서는 `[Console]::OutputEncoding = [System.Text.Encoding]::UTF8` 설정이 필수다. Claude Code는 WSL 환경에서 동작하므로 이 설정이 없으면 일본어(한국어 등 멀티바이트 문자)가 깨진다. `.env` 파일 파싱 시 `-split "=", 2`를 사용해 값 안에 `=`가 포함된 URL 등에도 대응한다.

Python 스크립트에서는 `sys.stdout.reconfigure(encoding="utf-8")`로 출력 인코딩을 지정하고, JSON 출력 시 `ensure_ascii=False`로 한국어·일본어 등을 유니코드 이스케이프 없이 출력한다. Excel 헤더를 고정값이 아닌 **동적 검출**로 읽어 열이 추가·삭제돼도 스크립트 수정이 불필요하게 설계한다.

---

**환경 구축 (요약):**

```bash
# 리포지터리 클론 및 .env 설정
git clone https://github.com/Masa1984a/claude-skills-demo.git
cd claude-skills-demo
cp .env.example .env
# .env 파일을 자신의 환경에 맞게 편집

# Python 라이브러리
pip install openpyxl playwright
python -m playwright install chromium

# PowerShell 모듈 (관리자 권한)
Install-Module -Name PnP.PowerShell -Scope CurrentUser -Force
```

---

## **정리 — Skills의 3가지 차별화 포인트:**
Progressive Disclosure에 의한 **컨텍스트 효율**, 로컬 실행에 의한 **클라우드 AI 제약 돌파**(브라우저 조작·사내 인증 등), 그리고 SKILL.md 공개 표준화에 의한 **이식성**이 기존 AI 에이전트와의 차별화 포인트다. Skills를 사용하면 "그런대로 우수한 어시스턴트"를 "믿을 수 있는 에이전트"로 육성할 수 있다.  