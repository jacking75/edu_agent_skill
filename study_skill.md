# skill 학습  


## 1단계: 개념과 배경 이해

### 1-1. Skill이란 무엇인가

Skill은 AI 에이전트에게 특정 작업을 수행하는 방법을 가르치는 **지침 폴더**입니다. 핵심 파일인 `SKILL.md` 하나와 선택적 보조 파일들로 구성됩니다. Anthropic의 공식 가이드는 이것을 "신입 사원에게 건네는 업무 인수인계서"에 비유합니다. 한 번 작성해두면 세션이 바뀌어도, 팀원이 프로젝트를 클론해도 동일한 작업 품질이 유지됩니다.

Skill이 없을 때는 매 대화마다 "우리 프로젝트는 Next.js 15를 쓰고, ESLint 설정은 이렇고…"라고 반복 설명해야 합니다. Skill이 있으면 에이전트가 해당 매뉴얼을 자동으로 읽고 정해진 절차와 스타일대로 작업합니다.

### 1-2. Skill이 등장한 배경 — 바이브 코딩의 품질 문제

2025년 2월 Andrej Karpathy가 "바이브 코딩"이라는 용어를 만들었습니다. AI에게 자연어로 지시하고 생성된 코드를 그대로 수용하는 방식인데, 이 방식이 확산되면서 코드 품질 문제가 데이터로 드러났습니다.

학습해야 할 핵심 연구 결과는 다음과 같습니다.

**GitClear 보고서 (2025)**: 211백만 줄의 코드를 분석한 결과, AI 도입 이후 코드 중복이 4배 증가했고, 역사상 처음으로 복사/붙여넣기 코드가 리팩토링 코드를 초과했습니다.

**Carnegie Mellon 연구**: 807개 GitHub 저장소에서 Cursor 도입 이후 정적 분석 경고가 30% 증가하고 코드 복잡도가 41% 증가했습니다. 속도 향상은 일시적이었지만 품질 문제는 누적되었습니다.

**Google DORA 보고서 (2025)**: AI 도입과 소프트웨어 전달 안정성 사이에 부정적 상관관계가 있었습니다. 핵심 결론은 "AI는 팀을 고치지 않는다. 이미 있는 것을 증폭시킨다"는 것이었습니다.

**Anthropic Agentic Coding Trends**: 개발자가 업무의 약 60%에 AI를 사용하지만, 완전히 위임할 수 있는 작업은 0~20%에 불과합니다. 나머지는 "사려 깊은 설정, 적극적인 감독, 인간의 판단"이 필요합니다.

이 "설정"과 "감독"을 코드화한 것이 바로 Skill입니다. AI가 빠르게 생산하는 코드에 품질 가드레일을 씌우는 역할을 합니다.

### 1-3. AI가 반복하는 전형적인 코드 문제 패턴

Skill이 해결하려는 구체적인 문제 유형을 이해해야 왜 특정 방식으로 Skill을 작성해야 하는지 알 수 있습니다.

**네이밍과 일관성 문제**: 같은 프로젝트 안에서 변수명이 camelCase와 snake_case가 혼용되거나, `data`, `tmp`, `proc` 같은 모호한 이름이 사용됩니다.

**코드 중복**: 공통 로직을 추출하지 않고 복사/붙여넣기합니다. 같은 계산이 여러 곳에 나타납니다.

**안전 검사 누락**: 입력 경계 검증 없음, 데이터 구조에 대한 가정, None/null 체크 누락이 발생합니다.

**가독성 문제**: 매직 넘버(예: 86400가 "하루의 초"라는 설명 없이 사용), 미사용 변수, 여러 추상화 수준이 섞인 함수가 등장합니다.

이 패턴들은 Robert C. Martin의 *Clean Code* 17장에서 정의한 66개 코드 스멜과 정확히 일치합니다. AI가 새로운 종류의 문제를 만드는 것이 아니라, 인간이 수십 년간 피하려 노력한 문제를 더 빠르고 대규모로 재생산하는 것입니다.

### 1-4. Rules vs Skills vs MCP — 세 개념의 차이

이 세 가지는 에이전트 생태계에서 서로 다른 역할을 하며 혼동하기 쉽습니다.

**Rules** (예: `.cursorrules`, `CLAUDE.md`의 규칙 섹션)는 **항상 켜져 있는 수동적 가드레일**입니다. 에이전트가 매 대화마다 자동으로 읽습니다. "코드에 항상 타입 힌트를 붙여라", "커밋 메시지는 Conventional Commits 형식을 따라라" 같은 전역 규칙에 적합합니다.

**Skills**는 **에이전트가 필요할 때만 활성화하는 능동적 지침**입니다. 사용자의 요청을 보고 description 필드와 의미적으로 매칭하여 관련 Skill만 로드합니다. "블로그 포스트를 작성해줘"라고 하면 블로그 작성 Skill이 활성화되지만, "이 버그를 고쳐줘"라고 하면 활성화되지 않습니다.

**MCP (Model Context Protocol)**는 에이전트가 외부 도구(GitHub, Notion, Slack, 데이터베이스 등)에 **연결하는 통로**입니다.

Anthropic의 공식 가이드는 이를 주방에 비유합니다. MCP는 재료와 도구가 갖춰진 전문 주방이고, Skills는 레시피입니다. MCP가 "무엇을 할 수 있는가(what Claude can do)"를 제공하고, Skills가 "어떻게 해야 하는가(how Claude should do it)"를 제공합니다.

### 1-5. Progressive Disclosure — Skill의 핵심 설계 원리

Skill의 아키텍처를 관통하는 가장 중요한 원리입니다. 모든 지침을 에이전트의 컨텍스트에 한꺼번에 넣으면 "Context Saturation"이 발생하여 성능이 저하됩니다. 대신 3단계로 나누어 필요한 만큼만 로드합니다.

**Level 1 — Discovery (항상 로드, ~100 토큰)**: 에이전트가 시작할 때 모든 Skill의 `name`과 `description` 프론트매터만 읽습니다. 이것만으로 "지금 이 요청에 어떤 Skill이 필요한가"를 판단합니다.

**Level 2 — Activation (트리거 시 로드, <5000 토큰 권장)**: 사용자의 요청이 특정 Skill의 description과 매칭되면, 그 Skill의 `SKILL.md` 본문 전체를 읽습니다. 핵심 지침, 단계별 절차, 예시가 여기에 있습니다.

**Level 3 — Execution (필요 시만 로드)**: `scripts/`, `references/`, `assets/` 폴더의 파일은 본문에서 참조할 때만 읽습니다. 상세 API 가이드, 검증 스크립트, 템플릿 등이 여기에 해당합니다.

이 구조 덕분에 수십 개의 Skill을 설치해도 에이전트가 느려지지 않습니다. React 컴포넌트를 작성할 때 데이터베이스 마이그레이션 Skill까지 읽을 필요가 없기 때문입니다.

### 1-6. Agent Skills Open Standard — 도구 간 호환성

Agent Skills는 Claude Code에서 시작했지만 이제 **오픈 표준**(https://agentskills.io)으로 공개되었습니다. 동일한 SKILL.md 파일이 Claude Code, Cursor, VS Code Copilot, Gemini CLI, OpenCode 등 여러 에이전트에서 작동합니다.

Vercel이 만든 **skills.sh** (https://skills.sh)는 이 표준을 기반으로 한 공개 마켓플레이스이며, `npx skills add` 명령 하나로 여러 에이전트에 동시 설치할 수 있습니다. 이것은 Skill을 "특정 도구에 종속된 설정"이 아니라 "팀의 지식 자산"으로 만드는 핵심 특성입니다.

### 1-7. Skill의 세 가지 유스케이스 카테고리

Anthropic이 관찰한 Skill 활용의 세 가지 패턴입니다.

**카테고리 1 — 문서/자산 생성**: 일관된 고품질 결과물(문서, 프레젠테이션, UI, 코드 등)을 만드는 Skill. 외부 도구 없이 AI의 내장 기능만 사용합니다. 예: `frontend-design` Skill, 블로그 포스팅 스타일 Skill.

**카테고리 2 — 워크플로우 자동화**: 일관된 방법론이 필요한 다단계 프로세스를 위한 Skill. 예: `skill-creator` Skill (Skill 작성 과정 자체를 자동화), 코드 리뷰 워크플로우 Skill.

**카테고리 3 — MCP 강화**: MCP가 제공하는 도구 접근에 워크플로우 지식을 더하는 Skill. MCP가 Linear에 연결해주면, Skill이 "스프린트 플래닝은 이 순서로 이렇게 해라"를 가르칩니다. 예: `sentry-code-review` Skill.

---

### 1단계 학습 자료

| 자료 | 내용 | 형식 |
|------|------|------|
| Anthropic 32페이지 가이드 PDF (https://resources.anthropic.com/hubfs/The-Complete-Guide-to-Building-Skill-for-Claude.pdf) | 1단계와 2단계의 거의 모든 내용을 포함하는 공식 자료. 가장 먼저 읽을 것 | PDF |
| Dev.to — "Skills, Not Vibes" (https://dev.to/gde/skills-not-vibes-teaching-ai-agents-to-write-clean-code-3l9e) | 품질 문제의 데이터, Clean Code와의 연결 | 영문 글 |
| Creative Tim 가이드 (https://www.creative-tim.com/blog/ai-agent/what-are-ai-agent-skills-a-practical-guide-to-skillmd) | skills.sh 생태계 전체 조감도 | 영문 글 |
| YouTube — "AI 에이전트 전용 Skills.md 훈련 가이드" (https://www.youtube.com/watch?v=g04eOqTdmz8) | 한국어로 전체 흐름 파악 | 한국어 영상 |
| YouTube — "클로드 스킬로 바이브 코딩 더 쉽게 하는 방법" (https://www.youtube.com/watch?v=rNtpNY41h5o) | 한국어 실습 중심 | 한국어 영상 |

---

## 2단계: 공식 스펙과 Best Practice 정독

### 2-1. 디렉토리 구조

Skill은 최소 `SKILL.md` 파일 하나를 포함하는 폴더입니다.

```
skill-name/
├── SKILL.md              ← 필수. 유일하게 반드시 있어야 하는 파일
├── scripts/              ← 선택. 에이전트가 실행할 수 있는 코드
│   ├── validate.py
│   └── check.sh
├── references/           ← 선택. 필요 시 읽는 추가 문서
│   ├── REFERENCE.md
│   └── api-guide.md
└── assets/               ← 선택. 템플릿, 이미지, 데이터 파일
    └── report-template.md
```

**폴더 이름 규칙**: 반드시 kebab-case를 사용합니다. `notion-project-setup`은 맞고, `Notion Project Setup`(공백), `notion_project_setup`(언더스코어), `NotionProjectSetup`(대문자)은 모두 틀립니다. 그리고 **폴더 이름은 SKILL.md 안의 `name` 필드와 정확히 일치**해야 합니다.

**README.md를 넣지 않습니다**: Skill 폴더 안에는 README.md를 두지 않습니다. 모든 문서는 SKILL.md 또는 references/에 넣습니다. (GitHub 저장소의 최상위에 사람을 위한 README는 별도로 둡니다.)

### 2-2. SKILL.md 파일의 두 부분

SKILL.md는 **YAML 프론트매터**와 **마크다운 본문**으로 구성됩니다.

```markdown
---
name: code-review-guide
description: Pull Request 코드 리뷰 자동화. PR, 코드 리뷰, 검토 요청 시 코드 품질, 보안, 스타일 검사 수행.
---

# 코드 리뷰 가이드

여기에 실제 지침이 들어갑니다...
```

프론트매터는 반드시 파일 최상단에 `---`로 열고 닫아야 합니다. 이 구분자가 없거나 따옴표가 닫히지 않으면 업로드 자체가 실패합니다.

### 2-3. 프론트매터 필드 상세 규칙

#### `name` (필수)

| 규칙 | 설명 |
|------|------|
| 글자 수 | 1~64자 |
| 허용 문자 | 유니코드 소문자 알파벳, 숫자, 하이픈(`a-z`, `0-9`, `-`)만 |
| 시작/끝 | 하이픈으로 시작하거나 끝날 수 없음 |
| 연속 하이픈 | `--` 불가 |
| 폴더명 일치 | 부모 폴더 이름과 정확히 같아야 함 |

```yaml
# 올바른 예시
name: pdf-processing
name: data-analysis
name: code-review

# 잘못된 예시
name: PDF-Processing      # 대문자 불가
name: -pdf                # 하이픈으로 시작 불가
name: pdf--processing     # 연속 하이픈 불가
name: my_cool_skill       # 언더스코어 불가
```

추가로, **"claude"나 "anthropic"이 포함된 이름은 예약어로 사용할 수 없습니다.**

#### `description` (필수)

이 필드가 Skill 작성에서 **가장 중요한 부분**입니다. 에이전트는 description만 보고 100개가 넘는 Skill 중에서 지금 사용할 것을 선택합니다. description이 모호하면 Skill이 아예 활성화되지 않습니다.

| 규칙 | 설명 |
|------|------|
| 글자 수 | 1~1024자 |
| 필수 내용 | "무엇을 하는가" + "언제 사용하는가" 둘 다 포함 |
| 금지 문자 | XML 꺾쇠괄호(`<`, `>`) 사용 금지 (보안상 이유 — 프론트매터가 시스템 프롬프트에 나타나므로) |

**description 작성 공식**: `[기능 설명] + [트리거 상황] + [핵심 키워드]`

```yaml
# 좋은 예시 — 구체적이고 트리거 키워드 포함
description: Analyzes Figma design files and generates developer handoff documentation. Use when user uploads .fig files, asks for "design specs", "component documentation", or "design-to-code handoff".

description: Pull Request 코드 리뷰 자동화. PR, 코드 리뷰, 검토, 피드백 요청 시 코드 품질, 보안, 성능, 스타일 자동 검사.

description: Manages Linear project workflows including sprint planning, task creation, and status tracking. Use when user mentions "sprint", "Linear tasks", "project planning", or asks to "create tickets".
```

```yaml
# 나쁜 예시 — 모호함
description: Helps with projects.
description: 코드 리뷰를 돕습니다.
description: Creates sophisticated multi-page documentation systems.
```

**description 자기 검증법**: 작성 후 에이전트에게 "When would you use the [skill name] skill?"이라고 물어봅니다. 에이전트가 description을 인용해서 답하므로, 그 답이 기대한 트리거 상황을 커버하는지 확인합니다.

**오버트리거링 방지 팁**: Skill이 관련 없는 요청에도 활성화되면, 부정 트리거를 추가합니다.

```yaml
description: Advanced data analysis for CSV files. Use for statistical modeling, regression, clustering. Do NOT use for simple data exploration (use data-viz skill instead).
```

#### `license` (선택)

```yaml
license: MIT
license: Apache-2.0
license: Proprietary. LICENSE.txt has complete terms
```

#### `compatibility` (선택, 1~500자)

환경 요구사항이 있을 때만 사용합니다.

```yaml
compatibility: Designed for Claude Code (or similar products)
compatibility: Requires git, docker, jq, and access to the internet
```

#### `metadata` (선택)

임의의 키-값 쌍을 넣을 수 있는 확장 필드입니다.

```yaml
metadata:
  author: my-team
  version: "1.0.0"
  mcp-server: linear
  category: productivity
  tags: [project-management, automation]
```

#### `allowed-tools` (선택, 실험적)

에이전트가 이 Skill 실행 중 사전 승인된 도구 목록입니다.

```yaml
allowed-tools: Bash(git:*) Bash(jq:*) Read
```

### 2-4. 마크다운 본문 작성법

프론트매터 아래의 본문이 실제 작업 지침입니다. 특별한 구문 제한은 없으며, 에이전트가 작업을 효과적으로 수행하는 데 도움이 되는 것이면 무엇이든 씁니다.

**Anthropic 권장 본문 구조 템플릿**:

```markdown
---
name: your-skill
description: [...]
---

# Skill 이름

## Instructions

### Step 1: [첫 번째 주요 단계]
구체적인 설명.

예시:
```bash
python scripts/fetch_data.py --project-id PROJECT_ID
```
기대 결과: [성공이 어떤 모습인지 설명]

### Step 2: [두 번째 주요 단계]
...

## Examples

### Example 1: [흔한 시나리오]
사용자가 말하는 것: "새 마케팅 캠페인을 설정해줘"
수행 절차:
1. MCP를 통해 기존 캠페인 조회
2. 제공된 파라미터로 새 캠페인 생성
결과: 확인 링크와 함께 캠페인 생성 완료

## Troubleshooting

### Error: [흔한 에러 메시지]
원인: [왜 발생하는지]
해결: [어떻게 고치는지]
```

**본문 작성의 핵심 원칙들**:

**구체적이고 실행 가능하게 쓸 것**: "데이터를 검증하세요"가 아니라 "`python scripts/validate.py --input {filename}`을 실행해서 데이터 포맷을 검사하세요. 실패하면 흔한 문제로는 필수 필드 누락(CSV에 추가), 날짜 형식 오류(YYYY-MM-DD 사용)가 있습니다"라고 씁니다.

**중요한 지침은 맨 위에**: 에이전트가 본문을 위에서 아래로 읽으므로, 핵심 규칙일수록 앞에 놓습니다. `## Important` 또는 `## Critical` 헤더를 사용하고, 정말 중요한 내용은 반복해도 됩니다.

**애매한 표현 금지**: "적절하게 처리하세요"가 아니라, "CRITICAL: `create_project`를 호출하기 전에 반드시 확인할 것 — 프로젝트 이름이 비어있지 않을 것, 팀 멤버가 최소 1명 배정되어 있을 것, 시작일이 과거가 아닐 것"으로 씁니다.

**나쁜 예와 좋은 예를 함께 보여줄 것**: 에이전트는 대조적인 예시에서 규칙을 가장 잘 학습합니다.

```markdown
# Bad - 매직 넘버 사용
if elapsed_time > 86400:
    ...

# Good - 명명된 상수 사용
SECONDS_PER_DAY = 86400
if elapsed_time > SECONDS_PER_DAY:
    ...
```

**선택지를 줄이고 명확한 권장을 할 것**: "pypdf, pdfplumber, PyMuPDF 중 하나를 사용할 수 있습니다"가 아니라 "텍스트 추출은 pdfplumber를 사용하세요. 스캔된 PDF에 OCR이 필요하면 pdf2image + pytesseract를 사용하세요"로 결정을 내려줍니다.

**검증 로직은 코드를 이용할 것**: 정말 중요한 검증은 언어 지침 대신 `scripts/` 안의 스크립트로 처리합니다. 자연어 해석은 확률적이지만 코드는 결정적입니다.

### 2-5. 본문 크기 관리와 파일 분리

| 항목 | 권장치 |
|------|--------|
| SKILL.md 본문 | 500줄 이하, 5,000토큰 이하 |
| 개별 참조 파일 | 각 파일을 집중적으로 유지 (작을수록 좋음) |
| 파일 참조 깊이 | SKILL.md에서 1단계 깊이만 |

본문이 길어지면 Progressive Disclosure 원칙에 따라 분리합니다.

```markdown
# SKILL.md 안에서 참조하는 방법

## SEO 최적화
상세한 SEO 가이드는 `references/seo-guide.md`를 참고하세요.

## 데이터 검증
검증이 필요하면 `scripts/validate.py`를 실행하세요.
```

**파일 참조 시 주의사항**: 스킬 루트로부터의 상대 경로를 사용합니다. `./reference.md`(현재 디렉토리 표기)나 `C:\docs\guide.md`(윈도우 절대경로) 대신 `references/guide.md`(Unix 스타일 상대경로)를 씁니다. 깊은 중첩 참조 체인(A가 B를 참조하고 B가 C를 참조)은 피합니다.

### 2-6. scripts/ 디렉토리 작성 규칙

스크립트는 에이전트가 직접 실행할 수 있는 코드입니다. Python, Bash, JavaScript가 흔히 사용됩니다.

**작성 시 지켜야 할 세 가지**: 자체 완결적이거나 의존성을 명확히 문서화할 것, 유용한 에러 메시지를 포함할 것, 엣지 케이스를 우아하게 처리할 것.

### 2-7. references/ 디렉토리 작성 규칙

에이전트가 필요할 때만 읽는 추가 문서입니다. `REFERENCE.md`(상세 기술 레퍼런스), `FORMS.md`(폼 템플릿), 도메인별 파일(`finance.md`, `legal.md` 등)을 넣을 수 있습니다. 각 파일은 하나의 주제에 집중하여 작게 유지합니다. 에이전트가 온디맨드로 로드하므로 파일이 작을수록 컨텍스트 소비가 적습니다.

### 2-8. 하나의 Skill = 하나의 명확한 역할

**틀린 예**: 하나의 Skill 안에 블로그 작성 + 코드 리뷰 + API 설계를 모두 넣기.

**맞는 예**: `blog-posting/`, `code-review/`, `api-design/`으로 분리하기.

범위가 넓은 Skill은 description이 모호해지고, 오버트리거링이 발생하며, 불필요한 지침이 컨텍스트를 낭비합니다.

### 2-9. 흔한 실수와 트러블슈팅

**Skill이 업로드 안 됨 — "Could not find SKILL.md"**: 파일 이름이 정확히 `SKILL.md`(대소문자 구분)여야 합니다. `SKILL.MD`, `skill.md`, `Skill.md` 모두 안 됩니다.

**Skill이 업로드 안 됨 — "Invalid frontmatter"**: YAML 구분자 `---`가 빠졌거나, 따옴표가 닫히지 않았거나, 들여쓰기가 잘못된 경우입니다.

```yaml
# 틀림 — 구분자 없음
name: my-skill
description: Does things

# 틀림 — 따옴표 미닫힘
---
name: my-skill
description: "Does things
---

# 맞음
---
name: my-skill
description: Does things
---
```

**Skill이 트리거 안 됨**: description이 너무 일반적이거나 트리거 키워드가 부족한 경우입니다. "프로젝트를 돕습니다"를 "Next.js 프로젝트 초기 설정 자동화. 프로젝트 생성, 초기 설정, scaffolding 요청 시 사용"으로 바꿉니다.

**Skill이 너무 자주 트리거됨**: description에 부정 트리거를 추가하거나 범위를 좁힙니다. "문서를 처리합니다"를 "PDF 법률 문서를 계약 검토용으로 처리합니다"로 구체화합니다.

**Skill이 로드됐지만 지침을 안 따름**: 지침이 너무 장황하거나(핵심이 묻힘), 표현이 애매하거나(구체적 행동 대신 추상적 표현), 중요한 내용이 본문 아래쪽에 묻혀 있는 경우입니다.

**컨텍스트 과부하로 응답이 느림**: SKILL.md가 너무 크거나, 동시에 활성화된 Skill이 너무 많은 경우입니다. 본문을 5,000 토큰 이내로 유지하고, 상세 내용을 references/로 분리합니다. 동시에 20~50개 이상의 Skill이 활성화되어 있다면 선택적으로 줄입니다.

### 2-10. Skill 검증

작성 후 공식 검증 도구로 스펙 준수 여부를 확인합니다.

```bash
skills-ref validate ./my-skill
```

이 명령은 프론트매터가 유효한지, 네이밍 규칙을 따르는지 등을 검사합니다.

### 2-11. 성공 기준 정의 방법

Skill을 만들기 전에 "잘 작동한다"의 기준을 먼저 세워야 합니다.

**정량적 기준** (대략적 벤치마크):

- 관련 쿼리 10~20개를 테스트하여 90% 이상에서 Skill이 자동 트리거되는가
- Skill 없이 작업할 때 대비 도구 호출 횟수와 토큰 소비가 줄어드는가
- 워크플로우 중 API 호출 실패가 0건인가

**정성적 기준**:

- 사용자가 다음 단계를 직접 프롬프트하지 않아도 에이전트가 진행하는가
- 사용자의 수정 없이 워크플로우가 완료되는가
- 동일 요청을 3~5회 반복해도 구조적으로 일관된 결과가 나오는가

---

### 2단계 학습 자료

| 자료 | 내용 | 우선순위 |
|------|------|----------|
| Agent Skills Specification (https://agentskills.io/specification) | 포맷의 절대적 기준. 모든 필드 규칙, 디렉토리 구조, Progressive Disclosure 정의 | 최우선 |
| Anthropic 32페이지 PDF (https://resources.anthropic.com/hubfs/The-Complete-Guide-to-Building-Skill-for-Claude.pdf) | Planning & Design, Testing & Iteration, Patterns & Troubleshooting 챕터 포함 | 최우선 |
| Anthropic Best Practices (https://platform.claude.com/docs/en/agents-and-tools/agent-skills/best-practices) | Progressive Disclosure 패턴, description 작성법, 본문 구조화 | 높음 |
| 한국어 완벽 가이드 (https://jeremyrecord.tistory.com/425) | 위 내용의 한국어 정리 + 실전 예시 | 높음 |
| Velog — Claude Agent Skills 사용방법 (https://velog.io/@yonghyuk/Claude-Agent-Skills-사용방법) | 3단계 로딩, 필드별 요구사항, 작성 팁 한국어 정리 | 보통 |

---

### 2단계 완료 후 자기 점검 체크리스트

2단계를 마쳤을 때 아래 질문에 모두 답할 수 있어야 합니다.

- SKILL.md의 두 부분(프론트매터 / 본문)의 역할 차이를 설명할 수 있는가
- `name` 필드의 5가지 제약 조건을 기억하는가 (길이, 허용 문자, 시작/끝, 연속 하이픈, 폴더명 일치)
- `description`에 반드시 포함해야 하는 두 가지 요소("무엇을 하는가" + "언제 사용하는가")를 알고 있는가
- 좋은 description과 나쁜 description의 차이를 3개 이상 예시로 들 수 있는가
- Progressive Disclosure의 3단계(메타데이터 → 본문 → 리소스 파일)를 그림으로 그릴 수 있는가
- `scripts/`, `references/`, `assets/`의 용도와 로딩 시점의 차이를 설명할 수 있는가
- SKILL.md 본문의 권장 크기 제한(500줄, 5000토큰)을 알고, 초과 시 어떻게 분리하는지 아는가
- `description`이 너무 모호할 때(언더트리거링)와 너무 넓을 때(오버트리거링)의 증상과 해결법을 아는가
- Skill 폴더 안에 README.md를 넣으면 안 되는 이유를 아는가
- 프론트매터에서 XML 꺾쇠괄호(`<`, `>`)가 금지된 보안상 이유를 이해하는가


---

# 3단계: SKILL.md 실전 작성 실습 가이드

> 이 가이드는 학생이 처음부터 끝까지 직접 따라할 수 있도록, 명령어 한 줄까지 구체적으로 안내합니다. 대상 에이전트는 **Claude Code**, **OpenCode**, **GitHub Copilot (VS Code)** 세 가지입니다.

---

## 사전 준비: 실습 환경 구성

실습을 시작하기 전에 세 가지 에이전트의 Skill 저장 경로를 정확히 이해해야 합니다. 동일한 SKILL.md 파일이 경로만 다를 뿐 세 도구 모두에서 작동합니다.

### 에이전트별 Skill 저장 경로

**Claude Code**는 프로젝트 수준에서 `.claude/skills/<skill-name>/SKILL.md`를, 개인 전역 수준에서 `~/.claude/skills/<skill-name>/SKILL.md`를 탐색합니다. Claude Code는 `.claude/skills/` 경로가 기본입니다. Skill이 설치되면 `/skill-name` 형태의 슬래시 명령으로 직접 호출할 수도 있고, description이 매칭되면 자동으로 활성화되기도 합니다.

**OpenCode**는 여러 경로를 동시에 탐색합니다. `.opencode/skills/<name>/SKILL.md`가 고유 경로이고, `.claude/skills/<name>/SKILL.md`와 `.agents/skills/<name>/SKILL.md`도 호환 경로로 인식합니다. 전역 경로는 `~/.config/opencode/skills/<name>/SKILL.md`, `~/.claude/skills/<name>/SKILL.md`, `~/.agents/skills/<name>/SKILL.md`입니다. OpenCode는 에이전트가 `skill({ name: "skill-name" })` 도구 호출로 Skill을 로드합니다.

**GitHub Copilot (VS Code)**는 프로젝트 수준에서 `.github/skills/<name>/SKILL.md`를, 개인 수준에서 `~/.copilot/skills/<name>/SKILL.md`를 탐색합니다. `.claude/skills/`와 `.agents/skills/`도 호환 경로로 인식합니다. VS Code 채팅창에서 `/skill-name`으로 슬래시 명령 호출이 가능합니다.

### 실습 프로젝트 생성

터미널에서 실습용 빈 프로젝트를 하나 만듭니다.

```bash
mkdir skill-lab && cd skill-lab
git init
mkdir -p .claude/skills
mkdir -p .opencode/skills
mkdir -p .github/skills
```

`.claude/skills/`에 Skill을 만들면 Claude Code와 OpenCode가 모두 인식합니다. GitHub Copilot을 위해서는 `.github/skills/`에도 같은 Skill을 복사하거나, `npx skills` CLI를 이용합니다.

---

## 실습 3-1: 기존 우수 Skill 분석 (2~3일)

직접 Skill을 만들기 전에, 잘 만들어진 Skill을 읽고 분석하는 것이 먼저입니다. "좋은 코드를 읽어야 좋은 코드를 쓴다"는 원리가 Skill에도 그대로 적용됩니다.

### 과제 A: anthropics/skills 저장소 클론 및 분석

```bash
git clone https://github.com/anthropics/skills.git ~/study/anthropics-skills
cd ~/study/anthropics-skills
```

이 저장소에서 두 가지 Skill을 집중 분석합니다.

**분석 대상 1: `skill-creator`**

이것은 "Skill을 만드는 방법을 가르치는 Skill"입니다. SKILL.md를 열고 다음 항목을 노트에 기록하세요.

```bash
cat skill-creator/SKILL.md
```

기록할 것: (1) description 필드에 어떤 트리거 키워드가 포함되어 있는가. (2) 본문이 몇 줄인가, 500줄 이내인가. (3) 본문의 섹션 구조는 어떤가 (헤더 계층이 어떻게 구성되었는가). (4) references/ 또는 scripts/ 폴더가 있는가, 있다면 SKILL.md에서 어떻게 참조하는가. (5) 예시(좋은 예/나쁜 예)가 포함되어 있는가.

**분석 대상 2: `frontend-design`**

```bash
cat frontend-design/SKILL.md
```

같은 항목을 기록하되, 추가로: (6) 이 Skill의 scope(담당 범위)는 어디까지인가, 코드 작성까지 포함하는가 아니면 디자인 원칙만 다루는가. (7) description이 카테고리 1(문서/자산 생성) 유형에 어떻게 맞춰져 있는가.

### 과제 B: skills.sh 인기 Skill 비교 분석

브라우저에서 https://skills.sh 에 접속하여 All Time 탭에서 상위 5개 Skill을 확인합니다.

각 Skill에 대해 다음 비교표를 작성하세요.

| Skill 이름 | description 길이 (대략) | 트리거 키워드 개수 | "무엇을" / "언제" 구분 여부 | 폴더 구조 (단일파일 vs 멀티파일) | 본문 내 예시 포함 여부 |
|---|---|---|---|---|---|
| (이름) | | | | | |

이 비교표를 통해 "잘 쓰인 description의 공통 패턴"을 발견하는 것이 목표입니다.

### 과제 C: Clean Code Skills 분석

```bash
git clone https://github.com/ertugrul-dmr/clean-code-skills.git ~/study/clean-code-skills
cd ~/study/clean-code-skills
```

이 저장소에는 `clean-comments`, `clean-functions`, `clean-general` 등 여러 Skill이 있습니다. 하나의 큰 주제(Clean Code)를 여러 Skill로 **분리한 방식**에 주목하세요.

기록할 것: (1) 각 Skill의 scope가 어떻게 나뉘어 있는가. (2) 한 Skill 안에서 "규칙 → 나쁜 예시 코드 → 좋은 예시 코드" 패턴이 어떻게 반복되는가. (3) `boy-scout`이라는 오케스트레이터 Skill이 다른 Skill들과 어떤 관계인가.

### 과제 D: 분석 보고서 작성

위 A~C의 분석 결과를 종합하여, 자신만의 "좋은 SKILL.md 작성 체크리스트"를 10개 항목으로 정리합니다. 예를 들어: "description에는 반드시 동사로 시작하는 기능 설명 + 'Use when' 트리거 문장을 포함한다" 등. 이 체크리스트는 이후 모든 실습에서 자기 검증 도구로 사용합니다.

---

## 실습 3-2: 직접 작성 — 난이도별 5단계

### 연습 1 (입문): Git 커밋 메시지 규칙 Skill

**목표**: 가장 단순한 단일 파일 Skill을 만들고, 세 에이전트 모두에서 작동하는지 확인합니다.

**Step 1 — 폴더 생성**

```bash
cd ~/skill-lab
mkdir -p .claude/skills/commit-message
```

**Step 2 — SKILL.md 작성**

`nano .claude/skills/commit-message/SKILL.md` (또는 VS Code로 파일 생성)

아래 내용을 직접 타이핑합니다 (복사/붙여넣기 대신 직접 치면서 구조를 체득합니다).

```markdown
---
name: commit-message
description: Generate conventional commit messages following the Conventional Commits spec. Use when committing code, writing commit messages, or when user mentions "commit", "커밋", or "git commit".
---

# Conventional Commit Messages

## Format

Every commit message follows this structure:

```
<type>(<scope>): <description>

[optional body]

[optional footer]
```

## Types

- feat: A new feature
- fix: A bug fix
- docs: Documentation only changes
- style: Formatting, missing semi-colons, etc (no code change)
- refactor: Code change that neither fixes a bug nor adds a feature
- test: Adding or correcting tests
- chore: Changes to build process or auxiliary tools

## Rules

1. Subject line must not exceed 50 characters
2. Use imperative mood ("add" not "added" or "adds")
3. Do not end the subject line with a period
4. Separate subject from body with a blank line
5. Body should explain WHAT and WHY, not HOW

## Examples

### Good

```
feat(auth): add OAuth2 login with Google

Implements Google OAuth2 flow to reduce signup friction.
Closes #142
```

### Bad

```
updated stuff
```

```
Fixed the bug in login page and also changed some CSS and updated the readme.
```
```

**Step 3 — 세 에이전트에 배포**

Claude Code와 OpenCode는 `.claude/skills/` 경로를 자동 인식하므로 추가 작업이 필요 없습니다. GitHub Copilot을 위해 복사합니다.

```bash
mkdir -p .github/skills/commit-message
cp .claude/skills/commit-message/SKILL.md .github/skills/commit-message/SKILL.md
```

또는 `npx skills` CLI로 한 번에 설치하는 방법을 연습하려면, 이 Skill을 GitHub 저장소에 push한 뒤 아래 명령을 사용합니다.

```bash
npx skills add <your-github-id>/skill-lab --skill commit-message -a claude-code -a opencode -a github-copilot
```

**Step 4 — 테스트**

각 에이전트에서 다음 프롬프트를 입력하고, Skill이 트리거되는지 확인합니다.

```
# Claude Code에서 (터미널)
claude
> 이 변경사항에 대한 커밋 메시지를 만들어줘

# 또는 슬래시 명령으로 직접 호출
> /commit-message 로그인 기능 추가
```

```
# OpenCode에서 (터미널)
opencode
> 커밋 메시지를 작성해줘
```

```
# VS Code에서 (Copilot 채팅창)
> /commit-message feat 타입으로 커밋 메시지 만들어줘
```

**검증 체크리스트**: (1) 세 에이전트 모두에서 Conventional Commits 형식이 적용되었는가. (2) "커밋" 키워드 없이 "이 변경사항을 정리해줘"라고 했을 때도 트리거되는가 (안 되면 description에 키워드 추가). (3) 슬래시 명령 `/commit-message`가 작동하는가.

---

### 연습 2 (초급): 코드 리뷰 Skill

**목표**: 좀 더 복잡한 지침과 체크리스트를 포함하는 Skill을 만듭니다.

**Step 1 — 폴더 생성 및 SKILL.md 작성**

```bash
mkdir -p .claude/skills/code-review
```

`.claude/skills/code-review/SKILL.md`를 작성합니다.

```markdown
---
name: code-review
description: Perform structured code review on files or pull requests. Use when user asks for "code review", "리뷰", "PR review", "코드 검토", or wants feedback on code quality, security, and performance.
---

# Structured Code Review

## Process

### Step 1: Understand Context
Read the changed files and understand the purpose of the changes.

### Step 2: Check Code Quality
Review against these criteria:
- **Single Responsibility**: Each function does one thing
- **Naming**: Variables and functions have descriptive names
- **Duplication**: No copy-pasted code blocks
- **Complexity**: No deeply nested conditionals (max 3 levels)

### Step 3: Check Security
- No hardcoded API keys or secrets
- User input is validated and sanitized
- SQL queries use parameterized statements
- No sensitive data in logs

### Step 4: Check Performance
- No unnecessary loops or redundant calculations
- Database queries are optimized (no N+1 problems)
- Large data sets use pagination

### Step 5: Provide Feedback

Format your review as:

```
## Summary
[1-2 sentence overview]

## Strengths
- [specific positive observation]

## Issues Found

### [Critical/Warning/Suggestion] — [file:line]
**Problem**: [what's wrong]
**Why it matters**: [impact]
**Suggested fix**:
[code snippet]
```

## What NOT to Review
- Personal style preferences not in the project style guide
- Whitespace-only changes
- Auto-generated files
```

**Step 2 — 배포 및 테스트**

```bash
mkdir -p .github/skills/code-review
cp .claude/skills/code-review/SKILL.md .github/skills/code-review/SKILL.md
```

테스트 프롬프트 예시:

```
이 파일을 리뷰해줘: src/auth/login.py
```

```
PR #42의 변경사항을 코드 리뷰해줘
```

```
/code-review src/utils/helpers.js
```

**실습 과제**: 리뷰 결과의 형식이 SKILL.md에서 정의한 포맷(Summary → Strengths → Issues Found)을 따르는지 확인합니다. 만약 에이전트가 형식을 따르지 않으면, 본문의 지침을 더 강하게 ("CRITICAL: Always use the exact format below") 수정하고 다시 테스트합니다.

---

### 연습 3 (중급): 폴더 구조를 갖춘 API 설계 Skill

**목표**: `references/`와 `scripts/` 폴더를 활용하는 Progressive Disclosure 패턴을 연습합니다.

**Step 1 — 전체 폴더 구조 생성**

```bash
mkdir -p .claude/skills/api-design/{references,scripts}
```

**Step 2 — SKILL.md 작성 (핵심 지침만)**

`.claude/skills/api-design/SKILL.md`:

```markdown
---
name: api-design
description: Design RESTful APIs following project conventions. Use when creating new API endpoints, designing API contracts, or when user mentions "API", "endpoint", "REST", or "route design".
---

# RESTful API Design Guide

## Core Principles

1. Use nouns for resources, not verbs: `/users` not `/getUsers`
2. Use HTTP methods correctly: GET (read), POST (create), PUT (full update), PATCH (partial update), DELETE (remove)
3. Use plural nouns: `/users` not `/user`
4. Nest resources for relationships: `/users/{id}/orders`
5. Maximum nesting depth: 2 levels

## Response Format

All responses follow a consistent envelope:

```json
{
  "data": {},
  "meta": { "page": 1, "total": 100 },
  "errors": []
}
```

## Error Handling

Use standard HTTP status codes:
- 400: Bad Request (validation errors)
- 401: Unauthorized
- 403: Forbidden
- 404: Not Found
- 409: Conflict
- 500: Internal Server Error

For detailed naming conventions and advanced patterns, see [references/naming-conventions.md](references/naming-conventions.md).

To validate an OpenAPI spec, run the validation script: `scripts/validate-spec.sh`
```

**Step 3 — references/naming-conventions.md 작성 (상세 참고 문서)**

```markdown
# API Naming Conventions — Detailed Reference

## URL Naming Rules

### Query Parameters
- Use camelCase: `sortBy`, `pageSize`, `startDate`
- Use `snake_case` only if the project convention requires it
- Boolean filters: `isActive=true`, not `active=1`

### Path Parameters
- Use kebab-case for multi-word resources: `/user-profiles`
- Use singular for single-resource actions: `/users/{id}`

### Versioning
- Prefix with version: `/v1/users`, `/v2/users`
- Never break existing versions; deprecate and migrate

### Pagination
- Use `page` and `pageSize` query parameters
- Default page size: 20, maximum: 100
- Always return `meta.total` in response

### Filtering
- Simple: `?status=active`
- Multiple values: `?status=active,pending`
- Range: `?createdAfter=2025-01-01&createdBefore=2025-12-31`

### Sorting
- Single field: `?sortBy=createdAt&order=desc`
- Multiple fields: `?sortBy=createdAt:desc,name:asc`
```

**Step 4 — scripts/validate-spec.sh 작성 (검증 스크립트)**

```bash
#!/bin/bash
# Validates an OpenAPI spec file
# Usage: ./validate-spec.sh <path-to-spec.yaml>

SPEC_FILE="${1:?Usage: validate-spec.sh <spec-file>}"

if [ ! -f "$SPEC_FILE" ]; then
  echo "ERROR: File not found: $SPEC_FILE"
  exit 1
fi

echo "Validating: $SPEC_FILE"

# Check required fields
for field in openapi info paths; do
  if ! grep -q "^${field}:" "$SPEC_FILE" 2>/dev/null && \
     ! grep -q "\"${field}\":" "$SPEC_FILE" 2>/dev/null; then
    echo "WARNING: Missing required field: $field"
  fi
done

# Check for common issues
if grep -qE '(password|secret|api_key)' "$SPEC_FILE"; then
  echo "SECURITY WARNING: Possible sensitive data in spec file"
fi

echo "Validation complete."
```

```bash
chmod +x .claude/skills/api-design/scripts/validate-spec.sh
```

**Step 5 — 세 에이전트에 배포**

```bash
# GitHub Copilot용
cp -r .claude/skills/api-design .github/skills/api-design

# OpenCode는 .claude/skills/를 자동 인식하므로 추가 작업 불필요
```

**Step 6 — 테스트 시나리오**

시나리오 1 — 기본 트리거: `사용자 관리 API를 설계해줘`라고 요청합니다. SKILL.md의 Core Principles가 적용되어 `/users`, `/users/{id}` 형태의 엔드포인트가 나오는지 확인합니다.

시나리오 2 — 참조 파일 로드: `API의 쿼리 파라미터 네이밍 규칙은 어때야 하지?`라고 요청합니다. 에이전트가 `references/naming-conventions.md`를 읽어서 camelCase 규칙, pagination 기본값 등을 답하는지 확인합니다.

시나리오 3 — 스크립트 실행: `/api-design spec.yaml 파일을 검증해줘`라고 요청합니다. 에이전트가 `scripts/validate-spec.sh`를 실행하는지 확인합니다.

**실습 과제**: 만약 시나리오 2에서 에이전트가 참조 파일을 읽지 않고 SKILL.md 본문만으로 답한다면, 본문의 참조 문구를 더 명시적으로 수정합니다. 예: "상세 네이밍 규칙은 반드시 `references/naming-conventions.md`를 읽은 후 답하세요."

---

### 연습 4 (심화): 프로젝트 전용 Skill 세트 — 3개 Skill의 역할 분리

**목표**: 하나의 프로젝트에서 여러 Skill이 scope 충돌 없이 각자의 역할을 수행하도록 설계합니다.

가상 프로젝트 설정: Next.js + TypeScript + Tailwind CSS로 만드는 웹 애플리케이션을 가정합니다.

**Skill 세트 설계**

| Skill 이름 | 담당 범위 | 트리거 상황 |
|---|---|---|
| `nextjs-component` | React 컴포넌트 작성 규칙 | 새 컴포넌트 생성, UI 개발 |
| `nextjs-api-route` | API Route 작성 규칙 | 서버 엔드포인트 개발 |
| `nextjs-testing` | 테스트 코드 작성 규칙 | 테스트 작성, TDD |

핵심은 **description이 서로 겹치지 않도록 설계**하는 것입니다.

**Skill 1: `.claude/skills/nextjs-component/SKILL.md`**

```markdown
---
name: nextjs-component
description: Create React components for Next.js with TypeScript and Tailwind CSS. Use when building UI components, pages, or layouts. Triggers on "component", "컴포넌트", "page", "layout", "UI".
---

# Next.js Component Guide

## File Structure
- Components go in `src/components/`
- Pages go in `src/app/`
- Use PascalCase for component files: `UserProfile.tsx`

## Component Template

```tsx
interface Props {
  // Always define explicit props interface
}

export function ComponentName({ prop1, prop2 }: Props) {
  return (
    <div className="...">
      {/* Tailwind classes only, no inline styles */}
    </div>
  );
}
```

## Rules
1. Always use named exports, not default exports
2. Props interface must be defined above the component
3. Use Tailwind CSS classes exclusively — no CSS modules, no inline styles
4. Server Components by default; add "use client" only when needed
5. Keep components under 150 lines; extract sub-components if longer
```

**Skill 2: `.claude/skills/nextjs-api-route/SKILL.md`**

```markdown
---
name: nextjs-api-route
description: Create Next.js API route handlers with proper validation and error handling. Use when building server endpoints, API routes, or backend logic. Triggers on "API route", "endpoint", "서버", "handler", "route handler".
---

# Next.js API Route Guide

## File Structure
- API routes go in `src/app/api/`
- Use route.ts naming convention

## Route Handler Template

```typescript
import { NextRequest, NextResponse } from 'next/server';
import { z } from 'zod';

const requestSchema = z.object({
  // Always validate input with zod
});

export async function POST(request: NextRequest) {
  try {
    const body = await request.json();
    const validated = requestSchema.parse(body);
    // ... business logic
    return NextResponse.json({ data: result });
  } catch (error) {
    if (error instanceof z.ZodError) {
      return NextResponse.json({ errors: error.issues }, { status: 400 });
    }
    return NextResponse.json({ errors: ['Internal server error'] }, { status: 500 });
  }
}
```

## Rules
1. Always validate input with zod schemas
2. Use try/catch with specific error types
3. Return consistent response envelope: `{ data, errors }`
4. Never expose internal error messages to clients
```

**Skill 3: `.claude/skills/nextjs-testing/SKILL.md`**

```markdown
---
name: nextjs-testing
description: Write tests for Next.js applications using Vitest and Testing Library. Use when creating tests, writing test cases, or debugging test failures. Triggers on "test", "테스트", "TDD", "vitest", "testing".
---

# Next.js Testing Guide

## Framework
- Unit tests: Vitest
- Component tests: @testing-library/react
- E2E tests: Playwright

## Test File Convention
- Co-locate test files: `ComponentName.test.tsx`
- Test directory for integration: `__tests__/`

## Component Test Template

```tsx
import { render, screen } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { ComponentName } from './ComponentName';

describe('ComponentName', () => {
  it('renders correctly with required props', () => {
    render(<ComponentName title="Test" />);
    expect(screen.getByText('Test')).toBeInTheDocument();
  });

  it('handles user interaction', async () => {
    const user = userEvent.setup();
    const onClickMock = vi.fn();
    render(<ComponentName onClick={onClickMock} />);
    await user.click(screen.getByRole('button'));
    expect(onClickMock).toHaveBeenCalledOnce();
  });
});
```

## Rules
1. Test behavior, not implementation details
2. Use `screen.getByRole` over `getByTestId` when possible
3. Each test should be independent — no shared mutable state
4. Name tests as sentences: "it renders correctly when..."
```

**배포**

```bash
# 세 Skill 모두 GitHub Copilot 경로에도 복사
for skill in nextjs-component nextjs-api-route nextjs-testing; do
  mkdir -p .github/skills/$skill
  cp .claude/skills/$skill/SKILL.md .github/skills/$skill/SKILL.md
done
```

**테스트 — Scope 분리 검증**

이것이 이 연습의 핵심입니다. 각 프롬프트가 **정확히 하나의 Skill만** 트리거하는지 확인합니다.

| 테스트 프롬프트 | 기대 트리거 Skill | 트리거 안 되어야 할 Skill |
|---|---|---|
| "UserProfile 컴포넌트를 만들어줘" | nextjs-component | nextjs-api-route, nextjs-testing |
| "/api/users POST 엔드포인트를 만들어줘" | nextjs-api-route | nextjs-component, nextjs-testing |
| "UserProfile 컴포넌트의 테스트를 작성해줘" | nextjs-testing | nextjs-component, nextjs-api-route |
| "로그인 페이지를 만들어줘" | nextjs-component | 나머지 |
| "이 API의 입력 검증을 추가해줘" | nextjs-api-route | 나머지 |

만약 "로그인 페이지를 만들어줘"에서 nextjs-api-route도 함께 트리거된다면, 해당 Skill의 description에서 "page"라는 키워드를 제거하고 더 구체적인 트리거 ("API route", "endpoint", "route handler")만 남깁니다.

**실습 과제**: Claude Code에서 `What skills are available?`이라고 물어보면 현재 로드된 모든 Skill의 description을 보여줍니다. 이를 통해 세 Skill이 모두 인식되는지, description이 서로 구분 가능한지 확인합니다.

---

### 연습 5 (심화+): Subagent 실행과 동적 컨텍스트 주입 (Claude Code 전용)

**목표**: Claude Code의 고급 기능인 `context: fork`와 `!`backtick 명령`을 활용하는 Skill을 만듭니다. 이 기능은 Claude Code에만 있는 확장이므로 OpenCode/Copilot에서는 fork 기능 없이 일반 Skill로 동작합니다.

```bash
mkdir -p .claude/skills/pr-summary
```

`.claude/skills/pr-summary/SKILL.md`:

```markdown
---
name: pr-summary
description: Summarize a GitHub pull request with diff analysis. Use when reviewing PRs or when user asks for "PR summary", "PR 요약", or "pull request review".
context: fork
agent: Explore
allowed-tools: Bash(gh *)
disable-model-invocation: true
---

## Pull Request Context

- PR diff: !`gh pr diff $ARGUMENTS`
- PR description: !`gh pr view $ARGUMENTS`
- Changed files: !`gh pr diff $ARGUMENTS --name-only`

## Your Task

Based on the PR context above:

1. **Summary**: Describe the overall purpose in 2-3 sentences
2. **Key Changes**: List the most important modifications by file
3. **Risk Assessment**: Identify potential issues (breaking changes, missing tests, security concerns)
4. **Review Recommendation**: Approve, Request Changes, or Comment — with reasoning
```

**테스트**

```bash
# Claude Code에서
claude
> /pr-summary 42
```

이 Skill은 `!`backtick` 구문으로 `gh pr diff 42` 등의 명령을 먼저 실행하고, 그 출력을 Claude에게 전달합니다. `context: fork`이므로 별도의 서브에이전트 컨텍스트에서 실행되어, 현재 대화를 방해하지 않습니다.

`disable-model-invocation: true`이므로 Claude가 자동으로 이 Skill을 사용하지 않고, 반드시 `/pr-summary` 슬래시 명령으로 직접 호출해야 합니다.

---

## 추가 실습 과제 (심화 학습)

### 추가 실습 A: 기존 Skill을 npx skills로 설치하고 구조 분석

커뮤니티에서 만든 Skill을 실제로 설치해보고, 설치된 파일 구조를 분석합니다.

```bash
cd ~/skill-lab

# Vercel의 React best practices Skill 설치 (세 에이전트 동시)
npx skills add vercel-labs/agent-skills --skill vercel-react-best-practices -a claude-code -a opencode -a github-copilot

# Anthropic의 frontend-design Skill 설치
npx skills add anthropics/skills --skill frontend-design -a claude-code -a opencode -a github-copilot
```

설치 후 각 경로에 파일이 어떻게 배치되었는지 확인합니다.

```bash
find .claude/skills -name "SKILL.md" -exec echo "=== {} ===" \; -exec head -10 {} \;
find .github/skills -name "SKILL.md" -exec echo "=== {} ===" \; -exec head -10 {} \;
find .opencode/skills -name "SKILL.md" -exec echo "=== {} ===" \; -exec head -10 {} \;
```

**실습 과제**: 설치된 Skill의 SKILL.md를 열어서, 자신이 연습 1~4에서 작성한 Skill과 비교합니다. description의 구체성, 본문의 구조, 예시의 양에서 어떤 차이가 있는지 기록합니다.

---

### 추가 실습 B: skill-creator 메타 Skill을 이용한 Skill 생성

Claude에게 Skill 작성을 맡겨보고, 그 결과물을 직접 검토하고 개선합니다.

**Step 1 — skill-creator 설치**

```bash
npx skills add anthropics/skills --skill skill-creator -a claude-code
```

**Step 2 — Claude에게 Skill 생성 요청**

```
/skill-creator 우리 프로젝트는 Python FastAPI 백엔드인데, 
새로운 API 엔드포인트를 만들 때마다 같은 패턴을 반복해. 
Pydantic 모델 정의, 라우터 함수 작성, 에러 핸들링, 
그리고 pytest 테스트까지 일관된 방식으로 만드는 Skill을 생성해줘.
```

**Step 3 — Claude가 생성한 SKILL.md 검토**

자신이 3-1에서 작성한 "좋은 SKILL.md 작성 체크리스트" 10개 항목으로 Claude가 만든 Skill을 평가합니다. 부족한 부분을 직접 수정합니다.

**Step 4 — 수정한 Skill을 세 에이전트에 설치하고 테스트**

```bash
# Claude가 생성한 Skill 폴더를 적절한 경로에 배치
cp -r <generated-skill-folder> .claude/skills/fastapi-endpoint
cp -r <generated-skill-folder> .github/skills/fastapi-endpoint
```

---

### 추가 실습 C: 에이전트별 동작 차이 비교 실험

동일한 Skill을 세 에이전트에 넣고, 같은 프롬프트를 던져서 **결과가 어떻게 다른지** 비교합니다.

**실험 설계**

연습 2에서 만든 `code-review` Skill을 사용합니다. 임의의 Python 파일 하나를 준비합니다.

```python
# test_target.py — 일부러 문제를 포함한 코드
import os
import sys
import json  # unused import

API_KEY = "sk-abc123xyz789"  # hardcoded secret

def proc(d, f=False):
    x = []
    for i in d:
        if f:
            if i['type'] == 'A':
                x.append(i['val'] * 1.0825)
            elif i['type'] == 'B':
                x.append(i['val'] * 1.05)
        else:
            x.append(i['val'])
    return x
```

**세 에이전트에서 동일 프롬프트 실행**

```
이 파일을 코드 리뷰해줘: test_target.py
```

**비교 기록표 작성**

| 비교 항목 | Claude Code | OpenCode | GitHub Copilot |
|---|---|---|---|
| Skill 자동 트리거 여부 | | | |
| 리뷰 결과 포맷이 SKILL.md 지침을 따르는가 | | | |
| 보안 이슈(하드코딩된 API_KEY) 발견 여부 | | | |
| 미사용 import 감지 여부 | | | |
| 네이밍 문제(proc, d, f, x, i) 지적 여부 | | | |
| 매직 넘버(1.0825, 1.05) 지적 여부 | | | |

이 비교를 통해 각 에이전트가 Skill의 지침을 얼마나 충실히 따르는지 차이를 체감할 수 있고, description이나 본문을 에이전트별로 미세 조정해야 할 수도 있다는 점을 배웁니다.

---

### 추가 실습 D: OpenCode 권한 제어 실습

OpenCode에만 있는 Skill 권한 제어 기능을 실습합니다. 프로젝트 루트에 `opencode.json`을 만들어서 특정 Skill의 접근을 제어합니다.

```json
{
  "permission": {
    "skill": {
      "*": "allow",
      "pr-summary": "ask",
      "nextjs-*": "allow"
    }
  }
}
```

이 설정에서 `pr-summary` Skill은 에이전트가 로드하려 할 때 사용자 승인을 요청하고, `nextjs-`로 시작하는 Skill은 자동 허용됩니다.

테스트: OpenCode에서 PR 요약을 요청했을 때 승인 프롬프트가 나타나는지, Next.js 관련 요청에서는 즉시 Skill이 로드되는지 확인합니다.

---

### 추가 실습 E: GitHub Copilot 고유 기능 — user-invokable / disable-model-invocation 조합 실험

GitHub Copilot(VS Code)에서는 Skill의 호출 방식을 세밀하게 제어할 수 있습니다. 네 가지 조합을 직접 만들어 차이를 체험합니다.

`.github/skills/` 아래에 4개의 테스트 Skill을 만듭니다.

**test-both-default** (기본: 슬래시 명령 + 자동 트리거 모두 가능)

```markdown
---
name: test-both-default
description: Test skill with default invocation settings. Use when user asks about "invocation test default".
---
This is the default behavior skill. Both slash command and auto-trigger work.
```

**test-manual-only** (수동 호출만 가능, 자동 트리거 안 됨)

```markdown
---
name: test-manual-only
description: Test skill that only triggers manually. Use when user asks about "invocation test manual".
disable-model-invocation: true
---
This skill only responds to /test-manual-only slash command.
```

**test-auto-only** (자동 트리거만, 슬래시 메뉴에 안 보임)

```markdown
---
name: test-auto-only
description: Test skill that auto-triggers but is hidden from menu. Use when user asks about "invocation test auto".
user-invokable: false
---
This skill loads automatically when relevant but is not in the / menu.
```

**test-disabled** (둘 다 비활성)

```markdown
---
name: test-disabled
description: Test skill that is fully disabled.
disable-model-invocation: true
user-invokable: false
---
This skill should never activate.
```

**실험**: VS Code Copilot 채팅에서 `/`를 입력하여 슬래시 메뉴에 어떤 Skill이 나타나는지 확인합니다. 그리고 "invocation test default에 대해 알려줘"처럼 각 description의 키워드를 포함한 프롬프트를 던져 자동 트리거 여부를 확인합니다.

---

## 실습 3-3: 테스트 및 피드백 루프 — 체계적 방법

모든 연습이 끝난 후, 아래의 표준 테스트 프로세스를 모든 Skill에 적용합니다.

### 트리거 테스트

각 Skill마다 "트리거 되어야 할 프롬프트" 5개와 "트리거 되면 안 되는 프롬프트" 5개를 작성합니다.

```
# commit-message Skill
## Should trigger:
1. "커밋 메시지 만들어줘"
2. "이 변경사항을 커밋하고 싶어"
3. "git commit message for this change"
4. "컨벤셔널 커밋 형식으로 정리해줘"
5. "/commit-message 로그인 기능 추가"

## Should NOT trigger:
1. "이 코드를 리뷰해줘"
2. "새 컴포넌트를 만들어줘"
3. "API 엔드포인트를 설계해줘"
4. "테스트를 작성해줘"
5. "프로젝트 구조를 설명해줘"
```

세 에이전트 모두에서 10개 프롬프트를 실행하고 결과를 기록합니다. 90% 이상 정확하면 합격, 미달이면 description을 수정하고 재테스트합니다.

### 기능 테스트

Skill이 트리거된 후 출력이 지침을 따르는지 확인합니다. SKILL.md에서 정의한 형식, 규칙, 체크리스트가 결과물에 반영되었는지 3회 반복 실행하여 일관성을 봅니다.

### 스펙 검증

```bash
# Agent Skills 공식 검증 도구 사용
npx skills-ref validate .claude/skills/commit-message
npx skills-ref validate .claude/skills/code-review
npx skills-ref validate .claude/skills/api-design
```

### 피드백 반영 사이클

테스트 결과를 바탕으로 SKILL.md를 수정합니다. 수정 → 테스트 → 수정 → 테스트를 최소 3회 반복합니다. 이 반복 과정에서 "description의 키워드 한 단어 차이가 트리거 여부를 결정한다"는 감각이 생깁니다.

---

## 전체 실습 요약 및 체크리스트

| 실습 | 난이도 | 핵심 학습 포인트 | 완료 기준 |
|---|---|---|---|
| 3-1 과제 A~D | 분석 | 좋은 Skill의 패턴 인식 | 10개 항목 체크리스트 작성 완료 |
| 연습 1: commit-message | 입문 | 프론트매터 규칙, 3개 에이전트 배포 | 3개 에이전트 모두에서 트리거 확인 |
| 연습 2: code-review | 초급 | 지침 구체화, 출력 형식 제어 | 출력이 정의한 리뷰 포맷을 따름 |
| 연습 3: api-design | 중급 | references/scripts 활용, Progressive Disclosure | 참조파일 로드와 스크립트 실행 확인 |
| 연습 4: Skill 세트 3개 | 심화 | Scope 분리, 오버트리거링 방지 | 프롬프트별 정확히 1개 Skill만 트리거 |
| 연습 5: pr-summary | 심화+ | context:fork, 동적 컨텍스트 주입 | Claude Code에서 서브에이전트 실행 확인 |
| 추가 A: npx skills 설치 | 도구 | CLI 활용, 설치 구조 이해 | 커뮤니티 Skill 설치 및 구조 분석 완료 |
| 추가 B: skill-creator | 도구 | AI에게 Skill 초안 생성 위임 | 생성된 Skill 검토 및 수정 완료 |
| 추가 C: 에이전트별 비교 | 비교 | 동일 Skill의 에이전트별 동작 차이 | 비교 기록표 작성 완료 |
| 추가 D: OpenCode 권한 | OpenCode 특화 | 권한 제어 (allow/deny/ask) | opencode.json 설정 및 승인 프롬프트 확인 |
| 추가 E: Copilot 호출 제어 | Copilot 특화 | user-invokable, disable-model-invocation | 4가지 조합의 동작 차이 기록 |

연습 1부터 순서대로 진행하되, 각 연습을 완료할 때마다 3-3의 테스트 루프를 반드시 실행합니다. "만들고 끝"이 아니라 "만들고 → 테스트하고 → 고치고 → 다시 테스트"하는 사이클을 반복하는 것이 이 단계의 핵심입니다.


---

# 4단계: 생태계 활용과 지속 개선 — 실습 가이드

> 3단계에서 Skill을 만들고 테스트하는 법을 익혔다면, 4단계에서는 그 Skill을 **공유하고, 커뮤니티에서 가져다 쓰고, 팀 단위로 관리하고, 계속 발전**시키는 법을 배웁니다. 이 단계부터 Skill은 "개인의 메모"에서 "팀의 지식 자산"으로 성격이 바뀝니다.

---

## 실습 4-1: npx skills CLI 완전 정복

`npx skills`는 Vercel이 만든 오픈소스 CLI 도구로, Skill을 **검색, 설치, 업데이트, 삭제**하는 모든 작업을 하나의 명령 체계로 처리합니다. 40개 이상의 에이전트를 지원하며, 한 번의 명령으로 여러 에이전트에 동시 설치할 수 있습니다.

### 실습 A: 기본 명령어 7종 체험

실습용 프로젝트 디렉토리에서 각 명령을 직접 실행하고, 결과를 관찰합니다.

**명령 1 — 저장소의 Skill 목록 조회 (설치하지 않고 보기만)**

```bash
cd ~/skill-lab
npx skills add vercel-labs/agent-skills --list
```

이 명령은 해당 저장소에 있는 모든 Skill의 이름과 description을 보여줍니다. 어떤 Skill이 있는지 미리 파악할 때 사용합니다. 출력에서 각 Skill의 description이 어떤 패턴으로 작성되었는지 관찰하세요.

**명령 2 — 특정 Skill을 세 에이전트에 동시 설치**

```bash
npx skills add vercel-labs/agent-skills \
  --skill frontend-design \
  -a claude-code -a opencode -a github-copilot
```

설치 후 각 경로에 파일이 생성되었는지 확인합니다.

```bash
ls -la .claude/skills/frontend-design/SKILL.md
ls -la .agents/skills/frontend-design/SKILL.md  # opencode, github-copilot 공용
```

`npx skills`는 OpenCode와 GitHub Copilot의 경우 `.agents/skills/` 경로를 사용합니다. 이 경로는 두 에이전트 모두 호환 경로로 인식합니다.

**명령 3 — 설치된 Skill 목록 확인**

```bash
npx skills list
```

에이전트별로 분류된 설치 목록이 출력됩니다. 전역 Skill과 프로젝트 Skill이 별도로 표시되는 점을 확인하세요.

```bash
# 전역 Skill만 보기
npx skills ls -g

# 특정 에이전트의 Skill만 보기
npx skills ls -a claude-code
```

**명령 4 — 키워드로 Skill 검색**

```bash
npx skills find typescript
npx skills find react
npx skills find python
```

키워드 없이 실행하면 대화형(fzf 스타일) 검색 인터페이스가 열립니다.

```bash
npx skills find
```

검색 결과에서 관심 있는 Skill을 바로 설치할 수 있습니다. 검색 결과의 Skill들을 살펴보면서 다양한 커뮤니티 Skill의 주제와 범위를 파악하세요.

**명령 5 — Skill 업데이트 확인 및 실행**

```bash
# 업데이트 가능한 Skill이 있는지 확인
npx skills check

# 모든 Skill을 최신 버전으로 업데이트
npx skills update
```

Skill은 원본 GitHub 저장소가 업데이트되면 새 버전을 받을 수 있습니다. 이 기능이 왜 중요한지 생각해보세요 — Skill도 코드처럼 지속적으로 개선됩니다.

**명령 6 — Skill 삭제**

```bash
# 대화형으로 삭제할 Skill 선택
npx skills remove

# 특정 Skill을 특정 에이전트에서만 삭제
npx skills remove frontend-design --agent claude-code

# 모든 에이전트에서 특정 Skill 삭제
npx skills remove frontend-design --agent '*'
```

**명령 7 — 새 Skill 템플릿 생성**

```bash
# 현재 디렉토리에 SKILL.md 템플릿 생성
npx skills init my-new-skill
```

이 명령은 기본 프론트매터와 본문 구조가 포함된 SKILL.md 파일을 자동 생성합니다. 새 Skill을 만들 때 빈 파일에서 시작하지 않고 이 템플릿에서 시작하면 규칙을 놓칠 확률이 줄어듭니다.

### 실습 B: 전역(Global) vs 프로젝트(Project) 설치 차이 체험

같은 Skill을 전역과 프로젝트 두 곳에 설치하고, 어떤 차이가 있는지 직접 확인합니다.

```bash
# 프로젝트 수준 설치 (기본값)
npx skills add anthropics/skills --skill skill-creator -a claude-code

# 전역 수준 설치
npx skills add anthropics/skills --skill skill-creator -a claude-code -g
```

```bash
# 프로젝트 수준 확인
ls .claude/skills/skill-creator/SKILL.md

# 전역 수준 확인
ls ~/.claude/skills/skill-creator/SKILL.md
```

프로젝트 수준 Skill은 해당 프로젝트 디렉토리에서만 작동하고, Git에 커밋하면 팀원도 사용할 수 있습니다. 전역 Skill은 어떤 프로젝트에서든 항상 사용 가능합니다. 이 차이를 실제로 다른 디렉토리로 이동하여 에이전트에서 Skill 목록을 확인해보면서 체감하세요.

```bash
# skill-lab 안에서
cd ~/skill-lab
claude
> What skills are available?
# → 프로젝트 Skill + 전역 Skill 모두 표시

# 다른 프로젝트에서
cd ~/other-project
claude
> What skills are available?
# → 전역 Skill만 표시
```

### 실습 C: Symlink vs Copy 설치 방식 이해

`npx skills add`를 대화형으로 실행하면 "Symlink" 또는 "Copy" 방식을 선택할 수 있습니다.

```bash
npx skills add vercel-labs/agent-skills --skill web-design-guidelines
```

**Symlink(권장)**: 하나의 원본 파일을 여러 에이전트 경로에서 심볼릭 링크로 참조합니다. 원본을 수정하면 모든 에이전트에 즉시 반영됩니다. `npx skills update`가 이 방식에서 가장 잘 작동합니다.

**Copy**: 각 에이전트 경로에 독립적인 사본을 만듭니다. 심볼릭 링크가 지원되지 않는 환경(일부 Windows 설정)에서 사용합니다.

설치 후 확인합니다.

```bash
# symlink인지 확인
ls -la .claude/skills/web-design-guidelines/SKILL.md
# lrwxr-xr-x ... -> ../../.skills-cache/web-design-guidelines/SKILL.md 형태면 symlink
```

---

## 실습 4-2: 자신의 Skill을 GitHub에 공개하고 skills.sh에 등록하기

커뮤니티에서 Skill을 가져다 쓰기만 하는 것이 아니라, 자신이 만든 Skill을 공개하여 다른 사람이 `npx skills add`로 설치할 수 있게 합니다.

### 실습 D: Skill 공개 저장소 만들기

**Step 1 — 저장소 구조 설계**

3단계에서 만든 Skill 중 범용적인 것을 골라 공개 저장소를 만듭니다. 예를 들어 `commit-message`와 `code-review` Skill을 공개한다고 가정합니다.

```bash
mkdir ~/my-agent-skills && cd ~/my-agent-skills
git init
```

저장소 구조를 `npx skills`가 인식하는 표준 형식으로 구성합니다.

```
my-agent-skills/
├── README.md                          ← 사람을 위한 저장소 소개 (Skill 폴더 안이 아닌 최상위)
├── skills/
│   ├── commit-message/
│   │   └── SKILL.md
│   └── code-review/
│       └── SKILL.md
```

`npx skills` CLI는 `skills/` 디렉토리를 자동으로 탐색하므로 이 구조를 따르면 별도 설정 없이 발견됩니다.

**Step 2 — README.md 작성**

저장소 최상위 README.md는 **사람이 읽는 문서**입니다. Skill 폴더 안에는 README.md를 넣지 않는다는 규칙을 기억하세요.

```markdown
# My Agent Skills

Reusable AI agent skills for code quality and workflow automation.

## Available Skills

| Skill | Description |
|---|---|
| `commit-message` | Generate conventional commit messages |
| `code-review` | Structured code review with security and performance checks |

## Installation

```bash
# Install all skills to Claude Code, OpenCode, and GitHub Copilot
npx skills add <your-github-id>/my-agent-skills \
  -a claude-code -a opencode -a github-copilot

# Install a specific skill
npx skills add <your-github-id>/my-agent-skills --skill commit-message
```

## License

MIT
```

**Step 3 — Skill 파일 복사 및 정리**

```bash
mkdir -p skills/commit-message skills/code-review

# 3단계에서 만든 파일 복사
cp ~/skill-lab/.claude/skills/commit-message/SKILL.md skills/commit-message/SKILL.md
cp ~/skill-lab/.claude/skills/code-review/SKILL.md skills/code-review/SKILL.md
```

**Step 4 — 스펙 검증**

```bash
npx skills-ref validate skills/commit-message
npx skills-ref validate skills/code-review
```

**Step 5 — GitHub에 push**

```bash
git add .
git commit -m "feat: initial release of commit-message and code-review skills"
git remote add origin https://github.com/<your-github-id>/my-agent-skills.git
git push -u origin main
```

**Step 6 — 자신의 저장소에서 설치 테스트**

```bash
cd ~/skill-lab

# 자기 저장소에서 설치 테스트
npx skills add <your-github-id>/my-agent-skills --list
npx skills add <your-github-id>/my-agent-skills --skill commit-message -a claude-code
```

이것이 작동하면, 전 세계 누구든 같은 명령으로 여러분의 Skill을 설치할 수 있습니다.

**Step 7 — skills.sh 인덱싱**

skills.sh는 공개 GitHub 저장소에 있는 Skill을 자동으로 인덱싱합니다. 저장소가 public이고 `SKILL.md`가 표준 형식을 따르면, 시간이 지나면서 skills.sh 검색 결과에 나타나게 됩니다. `npx skills find` 검색에도 발견됩니다. 저장소 설명(About)에 "agent-skills"라는 태그를 추가하면 발견 가능성이 높아집니다.

---

## 실습 4-3: 커뮤니티 리소스 탐색 루틴 만들기

### 실습 E: 주간 Skill 탐색 루틴

매주 한 번, 약 30분 동안 다음 순서로 새로운 Skill과 패턴을 탐색하는 루틴을 만듭니다.

**Step 1 — skills.sh 트렌딩 확인 (10분)**

브라우저에서 https://skills.sh 에 접속합니다. **Trending (24h)** 탭과 **Hot** 탭을 확인합니다. 지난주에 없던 새로운 Skill이 있는지, 어떤 주제가 부상하고 있는지 파악합니다.

관찰 노트를 작성하세요.

```markdown
## 2026-02-20 주간 Skill 탐색

### 트렌딩 관찰
- [Skill 이름]: [한줄 설명] — [내 프로젝트에 적용 가능 여부]
- [Skill 이름]: [한줄 설명] — [description 작성에서 배울 점]

### 새로 발견한 패턴
- [예: scripts/ 안에 Python 린터를 넣는 패턴이 늘고 있음]
```

**Step 2 — 관심 Skill 설치 및 SKILL.md 분석 (10분)**

트렌딩에서 관심 있는 Skill 1~2개를 골라 설치하고 내용을 읽습니다.

```bash
npx skills add <owner>/<repo> --skill <skill-name> -a claude-code -a opencode -a github-copilot

# 설치된 SKILL.md 내용 확인
cat .claude/skills/<skill-name>/SKILL.md
```

3단계에서 만든 자기 체크리스트로 이 Skill을 평가합니다. 특히 자신의 Skill에 없는 좋은 패턴이 있으면 메모합니다.

**Step 3 — Reddit/GitHub 커뮤니티 확인 (10분)**

다음 경로를 확인합니다.

- https://www.reddit.com/r/ClaudeCode — "skill" 키워드로 검색
- https://www.reddit.com/r/vibecoding — Skill 관련 토론
- https://github.com/heilcheng/awesome-agent-skills — 새로 추가된 Skill 확인

유용한 토론이나 팁을 발견하면 자기 Skill 개선에 반영합니다.

### 실습 F: awesome-agent-skills 저장소 활용

```bash
git clone https://github.com/heilcheng/awesome-agent-skills.git ~/study/awesome-agent-skills
cd ~/study/awesome-agent-skills
```

이 저장소는 카테고리별로 정리된 Skill 큐레이션입니다. README를 읽으며 현재 생태계에 어떤 종류의 Skill이 있는지 전체 지도를 그립니다. 자신의 전문 분야에서 아직 Skill이 없는 빈 영역을 발견하면, 그것이 자신이 만들어서 기여할 Skill의 후보가 됩니다.

---

## 실습 4-4: Skill의 버전 관리와 팀 공유

Skill도 코드와 마찬가지로 버전 관리, 코드 리뷰, 점진적 개선이 필요합니다.

### 실습 G: Git 기반 Skill 버전 관리

**Step 1 — metadata에 version 추가**

3단계에서 만든 모든 Skill의 프론트매터에 `metadata.version`을 추가합니다.

```yaml
---
name: code-review
description: ...
metadata:
  author: your-name
  version: "1.0.0"
---
```

**Step 2 — 변경 이력을 커밋 메시지로 추적**

Skill을 수정할 때마다 의미 있는 커밋 메시지를 남깁니다.

```bash
# description 키워드 추가
git add .claude/skills/code-review/SKILL.md
git commit -m "fix(code-review): add Korean trigger keywords to description"

# 새 검사 항목 추가
git commit -m "feat(code-review): add accessibility check section"

# version bump
git commit -m "chore(code-review): bump version to 1.1.0"
```

**Step 3 — 변경 기록 확인**

```bash
git log --oneline -- .claude/skills/code-review/SKILL.md
```

이 이력을 통해 "왜 description을 이렇게 바꿨는지", "어떤 트리거 키워드를 추가했는지" 등을 추적할 수 있습니다.

### 실습 H: 팀 공유 워크플로우 시뮬레이션

혼자 실습하더라도 "팀원이 있다고 가정"하고 다음 과정을 진행합니다.

**Step 1 — Skill을 프로젝트에 커밋**

```bash
cd ~/skill-lab
git add .claude/skills/ .github/skills/
git commit -m "feat: add project skills for code-review and commit-message"
git push
```

프로젝트 수준의 `.claude/skills/`와 `.github/skills/`를 Git에 커밋하면, 저장소를 clone하는 모든 팀원이 동일한 Skill을 자동으로 사용합니다.

**Step 2 — Pull Request로 Skill 변경 리뷰**

Skill을 수정할 때 직접 main 브랜치에 푸시하지 않고, PR을 만듭니다.

```bash
git checkout -b improve/code-review-skill

# SKILL.md 수정
nano .claude/skills/code-review/SKILL.md

git add .
git commit -m "feat(code-review): add memory leak detection checklist"
git push origin improve/code-review-skill
# → GitHub에서 PR 생성
```

PR 설명에 "이 Skill 변경의 이유"와 "테스트 결과"를 기록합니다. 코드 리뷰처럼 Skill 리뷰를 받는 습관을 만듭니다.

**Step 3 — 팀 Skill 가이드라인 문서 작성**

프로젝트 루트에 `SKILLS-GUIDE.md`를 만들어 팀의 Skill 관리 규칙을 정합니다.

```markdown
# Team Skills Guide

## Our Skills

| Skill | Owner | Version | Last Updated |
|---|---|---|---|
| commit-message | @your-name | 1.0.0 | 2026-02-15 |
| code-review | @your-name | 1.1.0 | 2026-02-20 |
| nextjs-component | @teammate | 1.0.0 | 2026-02-18 |

## Rules
1. Every Skill change goes through PR review
2. description 변경 시 반드시 트리거 테스트 결과를 PR에 첨부
3. version은 Semantic Versioning을 따름
   - 트리거 키워드 추가/수정: patch (1.0.1)
   - 새 섹션 추가: minor (1.1.0)
   - 구조적 변경: major (2.0.0)
4. 하나의 Skill이 200줄을 넘으면 분리를 검토
```

---

## 실습 4-5: Skill 지속 개선 — 반복 실험

### 실습 I: description A/B 테스트

하나의 Skill에 대해 description을 두 가지 버전으로 만들어 어떤 것이 더 정확하게 트리거되는지 비교합니다.

**버전 A (간결형)**

```yaml
description: Code review for pull requests. Use when reviewing code or PRs.
```

**버전 B (키워드 풍부형)**

```yaml
description: Perform structured code review on files or pull requests. Checks code quality, security vulnerabilities, performance issues, and naming conventions. Use when user asks for "code review", "리뷰", "PR review", "코드 검토", "피드백", or wants feedback on code changes.
```

각 버전으로 다음 10개 프롬프트를 세 에이전트에서 테스트하고 트리거 성공률을 기록합니다.

```
1. "이 코드를 리뷰해줘"
2. "PR을 검토해줘"
3. "이 변경사항에 대한 피드백 줘"
4. "코드 품질을 확인해줘"
5. "보안 문제가 있는지 봐줘"
6. "이 함수 괜찮아 보여?"
7. "Code review please"
8. "Review this PR"
9. "이 파일 개선할 점이 있어?"
10. "커밋하기 전에 확인해줘"
```

| 프롬프트 | 버전 A (Claude Code) | 버전 A (OpenCode) | 버전 A (Copilot) | 버전 B (Claude Code) | 버전 B (OpenCode) | 버전 B (Copilot) |
|---|---|---|---|---|---|---|
| 1 | ✅/❌ | ✅/❌ | ✅/❌ | ✅/❌ | ✅/❌ | ✅/❌ |
| ... | | | | | | |

이 실험을 통해 description의 구체성이 트리거 정확도에 얼마나 영향을 미치는지 정량적으로 확인할 수 있습니다.

### 실습 J: Skill 분리 리팩토링

3단계 연습 4에서 만든 `nextjs-component` Skill이 시간이 지나면서 점점 커졌다고 가정합니다. 150줄이 넘어가면 분리를 검토해야 합니다.

**Before: 하나의 비대한 Skill**

```
nextjs-component/
└── SKILL.md  (200줄 — 컴포넌트 작성 + 스타일링 + 접근성 + 상태관리 규칙이 모두 포함)
```

**After: 역할별 분리**

```
nextjs-component/
└── SKILL.md  (80줄 — 컴포넌트 작성 핵심 규칙만)

nextjs-styling/
└── SKILL.md  (60줄 — Tailwind CSS 스타일링 규칙)

nextjs-a11y/
└── SKILL.md  (50줄 — 접근성 체크리스트)
```

분리 후 각 Skill의 description이 겹치지 않는지 테스트하고, 기존에 트리거되던 프롬프트가 여전히 올바른 Skill에 매칭되는지 확인합니다.

### 실습 K: 에이전트에게 Skill 개선 요청하기

Anthropic의 Best Practice 중 "Claude는 자기 자신의 지침을 개선하는 데 뛰어나다"라는 점을 활용합니다.

```bash
# Claude Code에서
claude

> 현재 code-review Skill을 읽고, 아래 관점에서 개선점을 제안해줘:
> 1. description이 트리거 키워드를 충분히 포함하는가
> 2. 본문의 지침이 모호하지 않은가
> 3. 예시가 충분한가
> 4. Progressive Disclosure 원칙에 맞게 분리되어 있는가
>
> 현재 Skill 파일: .claude/skills/code-review/SKILL.md
```

Claude의 제안을 검토한 뒤, 타당한 부분만 반영합니다. 모든 제안을 무비판적으로 수용하지 말고, 자신의 체크리스트와 대조하여 판단합니다.

동일한 요청을 OpenCode와 VS Code Copilot에서도 해보면, 각 에이전트가 Skill을 어떤 관점에서 평가하는지 비교할 수 있습니다.

---

## 추가 실습 과제

### 추가 실습 L: 도메인 특화 Skill 팩 만들기

자신의 전문 분야에 맞는 Skill 3~5개를 하나의 저장소로 묶어 "Skill 팩"을 만듭니다.

예를 들어 **"Python 백엔드 Skill 팩"**:

```
python-backend-skills/
├── README.md
├── skills/
│   ├── fastapi-endpoint/
│   │   ├── SKILL.md
│   │   ├── references/
│   │   │   └── pydantic-patterns.md
│   │   └── scripts/
│   │       └── generate-openapi.sh
│   ├── sqlalchemy-model/
│   │   └── SKILL.md
│   ├── pytest-testing/
│   │   ├── SKILL.md
│   │   └── references/
│   │       └── fixture-patterns.md
│   ├── docker-deploy/
│   │   └── SKILL.md
│   └── python-logging/
│       └── SKILL.md
```

각 Skill을 작성한 뒤, 다음 시나리오로 "팩 전체의 통합 테스트"를 합니다.

```
# 시나리오: 새 기능 개발 전체 흐름
1. "User 모델을 만들어줘" → sqlalchemy-model 트리거
2. "User CRUD API를 만들어줘" → fastapi-endpoint 트리거
3. "API 테스트를 작성해줘" → pytest-testing 트리거
4. "Docker로 배포 설정해줘" → docker-deploy 트리거
5. "로깅을 추가해줘" → python-logging 트리거
```

하나의 개발 흐름에서 다섯 개 Skill이 순차적으로 **각자의 역할에만** 트리거되는지 확인합니다.

### 추가 실습 M: 다른 사람의 Skill에 기여하기 (Open Source Contribution)

커뮤니티 Skill 저장소에 직접 기여하는 연습을 합니다.

**Step 1 — 기여할 저장소 선택**

skills.sh에서 자주 사용하는 Skill 중 개선할 점이 있는 것을 골라 Fork합니다.

```bash
# 예: clean-code-skills에 기여
gh repo fork ertugrul-dmr/clean-code-skills
cd clean-code-skills
```

**Step 2 — 개선 작업**

예를 들어 기존 Skill에 한국어 트리거 키워드를 추가하거나, 누락된 예시를 보충하거나, 새로운 카테고리의 Skill을 추가합니다.

```bash
git checkout -b feat/add-korean-triggers
# SKILL.md 수정
git commit -m "feat: add Korean trigger keywords to clean-comments skill"
git push origin feat/add-korean-triggers
# → GitHub에서 원본 저장소로 PR 생성
```

**Step 3 — PR 설명 작성**

```markdown
## What changed
Added Korean trigger keywords to the `clean-comments` skill description.

## Why
Korean-speaking developers should be able to trigger this skill with natural Korean phrases like "주석", "코멘트", "문서화".

## Testing
Tested with Claude Code, OpenCode, and GitHub Copilot.
- "주석 정리해줘" → ✅ triggers clean-comments
- "불필요한 코멘트 삭제해줘" → ✅ triggers clean-comments
```

### 추가 실습 N: OpenCode 고급 설정 — 에이전트별 Skill 권한 분리

OpenCode에서는 에이전트(plan, code 등)마다 다른 Skill 접근 권한을 설정할 수 있습니다. 이를 활용해 "계획 단계에서는 읽기 전용 Skill만, 구현 단계에서는 모든 Skill 허용"하는 워크플로우를 만듭니다.

프로젝트 루트에 `opencode.json`을 작성합니다.

```json
{
  "agent": {
    "plan": {
      "permission": {
        "skill": {
          "code-review": "allow",
          "api-design": "allow",
          "commit-message": "deny",
          "docker-deploy": "deny"
        }
      }
    },
    "code": {
      "permission": {
        "skill": {
          "*": "allow"
        }
      }
    }
  }
}
```

이 설정에서 plan 에이전트는 코드 리뷰와 API 설계 Skill은 사용할 수 있지만, 커밋이나 배포 같은 실행 Skill은 차단됩니다. code 에이전트는 모든 Skill을 사용할 수 있습니다.

테스트: OpenCode에서 plan 모드와 code 모드를 전환하며 각 Skill의 접근 가능 여부가 달라지는지 확인합니다.

### 추가 실습 O: GitHub Copilot 전용 — VS Code Extension에서 Skill 배포

GitHub Copilot은 VS Code Extension을 통해 Skill을 배포할 수 있습니다. Extension의 `package.json`에 `chatSkills`를 등록하면 됩니다.

**Step 1 — 간단한 Extension 프로젝트 생성**

```bash
mkdir my-skill-extension && cd my-skill-extension
npm init -y
mkdir -p skills/commit-message
cp ~/skill-lab/.claude/skills/commit-message/SKILL.md skills/commit-message/SKILL.md
```

**Step 2 — package.json에 chatSkills 등록**

```json
{
  "name": "my-skill-extension",
  "version": "1.0.0",
  "contributes": {
    "chatSkills": [
      {
        "path": "./skills/commit-message"
      }
    ]
  }
}
```

**Step 3 — VS Code에서 로컬 Extension 테스트**

VS Code Extension 개발 환경에서 F5를 눌러 디버그 모드로 실행하면, Copilot 채팅에서 해당 Skill이 나타나는지 확인할 수 있습니다.

이 패턴은 팀 전체에 Skill을 배포할 때 유용합니다. Extension을 VS Code Marketplace에 게시하면 팀원이 Extension 설치만으로 모든 Skill을 받을 수 있습니다.

### 추가 실습 P: Skill 성과 측정 대시보드 만들기

Skill이 실제로 얼마나 도움이 되는지 측정하는 간단한 기록 시스템을 만듭니다.

`SKILLS-METRICS.md` 파일을 프로젝트에 만들고 매주 업데이트합니다.

```markdown
# Skills Performance Metrics

## Week of 2026-02-17

### commit-message
- Times triggered: 23
- Times manually invoked: 5
- Times auto-triggered correctly: 18
- False triggers: 0
- User corrections needed: 2
- Satisfaction: 9/10

### code-review
- Times triggered: 12
- Times manually invoked: 3
- Times auto-triggered correctly: 8
- False triggers: 1 (triggered on "이 코드 설명해줘" — should not have)
- User corrections needed: 4
- Satisfaction: 7/10
- Action: description에서 "코드" 단독 키워드를 제거하고 "코드 리뷰", "코드 검토"로 구체화

### Improvements Made This Week
- code-review: description에 부정 트리거 추가 ("Do NOT use for code explanation")
- commit-message: body 섹션에 Breaking Change footer 예시 추가
```

이 기록을 4주간 누적하면, 어떤 Skill이 잘 작동하고 어떤 Skill이 개선이 필요한지, description 수정이 트리거 정확도에 어떤 영향을 미쳤는지 데이터로 확인할 수 있습니다.

---

## 전체 실습 요약

| 실습 | 핵심 학습 포인트 | 완료 기준 |
|---|---|---|
| A: CLI 기본 명령 7종 | add, list, find, check, update, remove, init | 모든 명령을 한 번씩 실행하고 결과 확인 |
| B: 전역 vs 프로젝트 | 설치 범위의 차이, 다른 디렉토리에서의 동작 | 다른 프로젝트에서 전역 Skill만 보이는 것 확인 |
| C: Symlink vs Copy | 설치 방식의 차이, 업데이트 동작 | symlink 확인 명령 실행 |
| D: Skill 공개 저장소 | 저장소 구조, README 작성, npx skills 호환성 | 자기 저장소에서 npx skills add 성공 |
| E: 주간 탐색 루틴 | 트렌딩 파악, 커뮤니티 동향 추적 | 첫 주간 탐색 노트 작성 |
| F: awesome-agent-skills | 생태계 전체 지도 파악, 기여 후보 발굴 | 빈 영역 1개 이상 식별 |
| G: Git 버전 관리 | metadata version, 의미 있는 커밋 메시지 | 3회 이상 Skill 수정 커밋 이력 |
| H: 팀 공유 워크플로우 | PR 기반 Skill 리뷰, SKILLS-GUIDE.md | 가이드 문서 작성 완료 |
| I: description A/B 테스트 | 구체성이 트리거 정확도에 미치는 영향 | 두 버전의 비교 결과표 작성 |
| J: Skill 분리 리팩토링 | 비대해진 Skill의 분리 기준과 방법 | 분리 후 트리거 정확도 유지 확인 |
| K: 에이전트에게 개선 요청 | AI를 활용한 Skill 메타 개선 | 3개 에이전트의 개선 제안 비교 |
| L: 도메인 Skill 팩 | 여러 Skill의 통합 설계와 scope 관리 | 연속 시나리오에서 5개 Skill 순차 트리거 성공 |
| M: 오픈소스 기여 | 커뮤니티 PR 작성, 테스트 결과 첨부 | PR 1개 이상 제출 |
| N: OpenCode 에이전트별 권한 | plan/code 모드별 Skill 접근 분리 | opencode.json 설정 후 모드별 동작 확인 |
| O: Copilot Extension 배포 | chatSkills 기반 Skill 배포 | 로컬 Extension에서 Skill 표시 확인 |
| P: 성과 측정 | 트리거 정확도, 유저 수정 횟수 추적 | 4주간 SKILLS-METRICS.md 누적 |

4단계는 "한 번 하고 끝"이 아니라 **지속적으로 반복하는 단계**입니다. 실습 E의 주간 탐색 루틴과 실습 P의 성과 측정을 매주 습관으로 유지하면, Skill 작성 능력이 매주 조금씩 향상됩니다. 특히 description A/B 테스트(실습 I)를 반복하면서 "이 키워드 하나가 트리거를 결정한다"는 감각이 쌓이는 것이 이 단계의 가장 중요한 성과입니다.