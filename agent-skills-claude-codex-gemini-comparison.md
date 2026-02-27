# Claude Code / Codex / Gemini CLI Agent Skills 비교 정리
  
[출처](https://zenn.dev/hiraoku/articles/e3f750a9fe96dc )  

---

## **공통 배경: Agent Skills 오픈 표준**
3가지 도구 모두 **Agent Skills** 오픈 표준([agentskills.io](https://agentskills.io/))을 따른다. 원래 Anthropic이 제창했으며 OpenAI Codex·Gemini CLI·Cursor 등 여러 도구가 채용했다. 핵심 포맷은 공통이며, `SKILL.md` 파일을 중심으로 한 디렉토리 구성을 취한다.

```
my-skill/
├── SKILL.md          # 필수: 메타데이터 + 지시사항
├── scripts/          # 선택: 실행 가능한 스크립트
├── references/       # 선택: 참조 문서
└── assets/           # 선택: 템플릿 등
```

**SKILL.md 기본 구조 (3가지 도구 공통):**

```markdown
---
name: skill-name
description: 스킬 설명. 언제 사용해야 하는지 명확하게 작성
---

# 스킬 지시 내용 (Markdown)

에이전트가 따라야 할 절차·규칙·예시를 여기에 기술
```

---

## **1. Claude Code**

**배치 위치:**

| 스코프 | 경로 |
|--------|------|
| 사용자 개인 | `~/.claude/skills/` |
| 프로젝트 | `.claude/skills/` |
| 구 커맨드 호환 | `.claude/commands/` (계속 동작) |

**호출 방법:**
- **명시적**: `/skill-name` 슬래시 커맨드로 직접 호출
- **암묵적**: Claude가 태스크 내용과 description을 대조해 자동 로딩

**핵심 기능:**

Progressive Disclosure(단계적 개시)를 채용해, 프론트매터만(~100토큰) 먼저 스캔하고 필요할 때만 전문(<5k토큰)을 로딩한다. 이를 통해 컨텍스트 윈도우를 절약한다.

`context: fork`로 서브에이전트 연계가 가능하며, 스킬 실행을 별도 에이전트(Explore 등)에 포크할 수 있다. `agent: Explore`처럼 에이전트 타입도 지정 가능하다.

구 `.claude/commands/`와 `.claude/skills/`가 통합되어 있으며, name이 그대로 `/커맨드명`이 된다. Claude.ai나 API에서도 ZIP 업로드 또는 Skills API(`/v1/skills`)로 관리 가능하다. docx, pdf, pptx, xlsx, skill-creator 등의 조합 스킬도 표준 탑재되어 있다.

**설정 예시:**

```markdown
---
name: deep-research
description: Research a topic thoroughly
context: fork
agent: Explore
---

Research $ARGUMENTS thoroughly:
1. Find relevant files using Glob and Grep
2. Read and analyze the code
3. Summarize findings
```

---

## **2. OpenAI Codex**

**배치 위치:**

| 스코프 | 경로 |
|--------|------|
| 사용자 개인 | `~/.codex/skills/` |
| 리포지토리 | `.agents/skills/` (CWD에서 리포지토리 루트까지 재귀 스캔) |
| 시스템 조합 | `~/.codex/skills/.system/` (plan, skill-creator 등) |

**호출 방법:**
- **명시적**: `/skills` 커맨드 또는 `$`로 스킬을 멘션
- **암묵적**: 태스크가 description에 매칭되면 자동 선택

**핵심 기능:**

프로젝트의 기본 지시는 `AGENTS.md`에, 전문 태스크는 Skills로 분리해서 관리한다. `AGENTS.md`는 Linux Foundation 산하 Agentic AI Foundation이 표준화한 파일이다.

`$skill-installer`라는 조합 스킬 인스톨러가 있어, 실험적 스킬이나 리모트 리포지토리에서 설치가 가능하다.

```bash
$skill-installer install the linear skill from the .experimental folder
```

`agents/openai.yaml`로 UI 메타데이터, 호출 정책, 툴 의존 관계를 YAML로 선언할 수 있다. `config.toml`로 스킬 개별 비활성화도 가능하다.

```toml
[[skills.config]]
path = "/path/to/skill/SKILL.md"
enabled = false
```

심볼릭 링크도 지원한다.

**AGENTS.md 계층 구조 (Codex 고유 강점):**

```
~/.codex/AGENTS.md                              # 글로벌
~/.codex/AGENTS.override.md                     # 글로벌 오버라이드
repo-root/AGENTS.md                             # 리포지토리
repo-root/services/payments/AGENTS.override.md  # 디렉토리 단위
```

CWD에 가까운 파일이 우선된다.

---

## **3. Gemini CLI**

**배치 위치:**

| 스코프 | 경로 |
|--------|------|
| 워크스페이스 | `.gemini/skills/` |
| 사용자 개인 | `~/.gemini/skills/` |
| 확장 기능 (Extension) | Extension에 번들 |

동일 이름 스킬이 있을 경우 우선순위는 **Workspace > User > Extension** 순이다.

**호출 방법:**
- **암묵적**: Gemini가 태스크를 인식하면 `activate_skill` 툴을 호출
- **사용자 승인**: 활성화 시 **확인 프롬프트**가 표시되어 이름·목적·접근 디렉토리를 확인 후 활성화 (동의형)

**핵심 기능:**

스킬 활성화 시 UI에서 확인 다이얼로그가 뜨는 **명시적 동의 모델**을 채용한다. 디렉토리 접근 권한도 명시된다. 한 번 활성화되면 세션 종료까지 지속된다.

`/skills` 커맨드로 스킬 관리가 가능하다.

```bash
/skills list            # 목록 표시
/skills enable <name>   # 활성화
/skills disable <name>  # 비활성화
--scope workspace       # 스코프 지정
```

Gemini CLI의 Extension 시스템으로 스킬을 패키지화·공유할 수 있다. 프로젝트 전체 컨텍스트는 `GEMINI.md`(또는 `AGENTS.md`)에, 전문 태스크는 Skills에 역할 분담한다. skill-creator가 조합 탑재되어 있어 "새 스킬을 만들어줘"라고 하면 자동으로 틀이 생성된다.

**GEMINI.md / AGENTS.md 대응:** Gemini CLI는 두 파일 모두 지원하며, 같은 디렉토리에 두 파일이 모두 있으면 `GEMINI.md`가 우선된다.

---

## **3가지 도구 비교표**

| 항목 | Claude Code | Codex | Gemini CLI |
|------|------------|-------|------------|
| **스킬 파일** | SKILL.md | SKILL.md | SKILL.md |
| **컨텍스트 파일** | CLAUDE.md | AGENTS.md | GEMINI.md / AGENTS.md |
| **개인 스킬 경로** | `~/.claude/skills/` | `~/.codex/skills/` | `~/.gemini/skills/` |
| **프로젝트 스킬 경로** | `.claude/skills/` | `.agents/skills/` | `.gemini/skills/` |
| **명시적 호출** | `/skill-name` | `/skills` 또는 `$` 멘션 | `/skills` 커맨드 |
| **암묵적 호출** | ✅ description 대조 | ✅ description 대조 | ✅ activate_skill 툴 |
| **활성화 시 동의** | 없음 (자동) | 없음 (자동) | **있음 (UI 확인)** |
| **서브에이전트 연계** | `context: fork` + agent 지정 | Agents SDK 연계 | — |
| **스킬 인스톨러** | — | `$skill-installer` | Extension 시스템 |
| **Progressive Disclosure** | ✅ | ✅ | ✅ |
| **조합 스킬** | docx, pdf, pptx, xlsx 등 | plan, skill-creator | skill-creator |
| **GUI (비CLI) 지원** | Claude.ai ZIP 업로드 | Codex App | Gemini Code Assist (IDE) |
| **오픈 표준** | Agent Skills 제창 원조 | Agent Skills 채용 | Agent Skills 채용 |

---

## **실전 포인트**

**스킬을 3가지 도구 간에 공유하는 방법:**

Agent Skills 오픈 표준을 준수하기 때문에 같은 `SKILL.md`를 여러 도구에서 사용할 수 있다. 심볼릭 링크로 공유 디렉토리를 활용하면 편리하다.

```bash
# 공통 스킬 디렉토리 생성
mkdir -p ~/.agent-skills/my-skill

# 각 도구에서 심볼릭 링크
ln -s ~/.agent-skills/my-skill ~/.claude/skills/my-skill
ln -s ~/.agent-skills/my-skill ~/.codex/skills/my-skill
ln -s ~/.agent-skills/my-skill ~/.gemini/skills/my-skill
```

**description 작성이 전부다:**

3가지 도구 모두 암묵적 호출은 **description 필드의 품질**에 달려 있다. 핵심 포인트는 언제 사용해야 하는지(트리거 조건)를 명확히 하고, 언제 사용하면 안 되는지의 경계도 기술하고, 200자 이내로 구체적으로 작성하는 것이다(Claude 제한 사항).

**컨텍스트 파일 vs 스킬 사용 구분:**

| 용도 | 컨텍스트 파일 | 스킬 |
|------|--------------|------|
| 코딩 규약 | ✅ | — |
| 테스트 실행 방법 | ✅ | — |
| 특정 워크플로우 절차 | — | ✅ |
| 도메인 전문 지식 | — | ✅ |
| 프로젝트 전체 규칙 | ✅ | — |
| 온디맨드로 사용하는 전문 태스크 | — | ✅ |    