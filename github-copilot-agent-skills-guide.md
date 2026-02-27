# GitHub Copilot Agent Skills (`SKILL.md`) 활용 가이드

[출처](https://qiita.com/TooMe/items/230a730ce0387c77e822 )  
  
**배경 및 기존 방식의 문제점:**

기존에는 `.github/copilot-instructions.md` 하나에 모든 프로젝트 컨텍스트(테스트 전략, 아키텍처, 코딩 규약 등)를 몰아넣었다. 이 방식의 단점은 명확하다. 모델 성능이 낮거나 대화가 길어지면 초반에 정의한 내용을 망각하고, 방대한 컨텍스트가 노이즈가 되어 추론 속도 저하 및 엉뚱한 답변을 유발한다.

---

## **Agent Skills란:**

2025년 12월, VSCode v1.108 업데이트에서 GitHub Copilot이 **Agent Skills**에 정식 대응했다. Agent Skills는 Copilot이 **필요할 때만 선택적으로 읽어들이는** 명령·리소스 파일 묶음이다. 각 스킬은 `SKILL.md`라는 파일로 정의하며, 프로젝트 도메인 지식을 분야별로 분리해서 관리하는 것이 핵심 사용 패턴이다.

---

## **`SKILL.md` 파일 구조:**

파일 상단에 YAML 형식의 프론트매터(frontmatter)를 작성한다.

```markdown
---
name: testing
description: テスト戦略。Vitest、React Testing Library、MSWを使用した単体・コンポーネントテストについて説明。テスト作成時に参照。
---

# 테스트 전략
...본문...
```

| 파라미터 | 설명 |
|---|---|
| `name` | 해당 `SKILL.md`가 위치한 디렉터리 이름 |
| `description` | 스킬의 기능과 **언제 참조할지**를 기술 → Copilot이 이걸 보고 필요 여부를 판단한다 |

`description`이 핵심이다. Copilot은 이 설명을 기반으로 현재 태스크와 관련된 `SKILL.md`만 선택적으로 로드한다.

---

## **디렉터리 구성:**

`.github/skills/` 하위에 **기능 영역별 디렉터리**를 만들고, 각 디렉터리 안에 `SKILL.md`를 배치한다.

```
.github
│── skills
│   ├── architecture
│   │   └── SKILL.md       # 파일 배치, 모듈 구성
│   ├── ui-design
│   │   └── SKILL.md       # Tailwind CSS, 컴포넌트 설계
│   ├── state-management
│   │   └── SKILL.md       # Zustand, React Query, React Hook Form
│   ├── backend-integration
│   │   └── SKILL.md       # Firebase Auth, Google Calendar API
│   ├── coding-standards
│   │   └── SKILL.md       # TypeScript/React 베스트 프랙티스, 네이밍
│   └── testing
│       └── SKILL.md       # Vitest, RTL, MSW, E2E
└── copilot-instructions.md
```

> **Tip:** VSCode의 `chat.agentSkillsLocations` 설정을 통해 임의의 디렉터리 구조도 지원한다.

---

## **`copilot-instructions.md`의 변화:**

`SKILL.md`로 세부 내용을 분리하면 `copilot-instructions.md`는 아래처럼 **매우 단순해진다.** 전체적인 전제 조건과 어떤 태스크에 어느 스킬 파일을 참조할지의 안내 정도만 남긴다.

```markdown
# Copilot Instructions

## 전제 조건
- 답변은 반드시 일본어로 할 것
- 변경량이 200줄을 초과할 가능성이 높으면 사전에 사용자에게 확인할 것
- 큰 변경 시 계획을 먼저 제시할 것

## 참조 스킬 가이드
- 아키텍처/디렉터리 구성 → `.github/skills/architecture/SKILL.md`
- UI 구현/스타일링     → `.github/skills/ui-design/SKILL.md`
- 상태 관리/데이터 페치 → `.github/skills/state-management/SKILL.md`
- 백엔드 연계/인증      → `.github/skills/backend-integration/SKILL.md`
- 코딩 규약             → `.github/skills/coding-standards/SKILL.md`
- 테스트 전략           → `.github/skills/testing/SKILL.md`
```

> **Note:** "이 태스크에는 이 SKILL.md를 참조해라"는 명시적 안내는 없어도 동작한다. Copilot이 `description`을 보고 자동으로 판단하기 때문이다. 안내를 남기는 것은 어디까지나 보험 차원이다.

---

## **도입 방법 (2단계):**

1. **VSCode 설정에서 `chat.useAgentSkills`를 `true`로 활성화**한다. (향후 기본값이 될 가능성이 있다.)
2. 위의 디렉터리 구조대로 **각 `SKILL.md`를 작성**한다.

---

## **정리하면:**

기존의 monolithic한 `copilot-instructions.md` 방식 대비, Agent Skills를 활용하면 컨텍스트를 필요한 시점에 필요한 만큼만 주입할 수 있다. 결과적으로 망각 문제가 줄고, 노이즈가 감소하며, Copilot의 응답 정확도와 속도가 향상된다.  