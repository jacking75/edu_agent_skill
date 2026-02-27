# Draw.io × GitHub Copilot / Claude Code — Skill + VS Code 확장으로 설계도를 Git 관리하는 법
  
[출처](https://tech-lab.sios.jp/archives/51560 )

---

## **문제 제기 — MCP Tool Server의 한계:**
Draw.io로 그린 아키텍처 도, 시퀀스 도, 플로우 도는 소스코드와 동일한 **설계 정보**다. 따라서 GitHub에 올려 버전 관리하고 PR로 리뷰해야 한다. 그런데 MCP Tool Server는 URL 방식이라, 그림을 생성하면 `app.diagrams.net`의 URL이 반환될 뿐 **로컬에 파일이 남지 않는다.** `git add` 할 대상이 없다. URL 공유·빠른 확인 용도에는 충분하지만, 설계 정보를 Git으로 관리하는 용도에는 맞지 않는다.

---

## **해결책 — Skill + VS Code 확장 조합:**
`jgraph/drawio-mcp` 리포지터리에는 `.drawio` 파일을 로컬에 생성하는 접근법이 함께 제공된다. 이것과 VS Code Draw.io 확장을 조합하면, **생성 → 프리뷰 → 편집 → 익스포트 → Git 관리까지 VS Code 안에서 완결**된다.

---

## **세팅 방법 (2단계):**

**① Skill 파일 배치**

MCP 서버 기동도, Node.js도 필요 없다. 아래 명령 한 줄로 끝난다.

```bash
mkdir -p .claude/skills/drawio
curl -sL https://raw.githubusercontent.com/jgraph/drawio-mcp/main/skill-cli/SKILL.md \
  > .claude/skills/drawio/SKILL.md
```

**② VS Code 확장 설치**

`hediet.vscode-drawio` 확장을 설치한다. devcontainer를 쓰는 경우 아래처럼 한 줄 추가면 끝이다.

```json
{
  "customizations": {
    "vscode": {
      "extensions": ["hediet.vscode-drawio"]
    }
  }
}
```

이 확장을 설치하면 `.drawio` 파일을 열었을 때 VS Code 안에 GUI 에디터가 표시된다. 드래그&드롭, 색 변경, 연결선 수정 등 브라우저의 draw.io와 동일한 조작이 가능하다. PNG·SVG 익스포트도 이 확장 메뉴에서 바로 된다.

---

## **왜 "CLI"가 아니라 "확장"인가:**
공식 리포지터리에서는 이 접근법을 "Skill + CLI"라고 부른다. 여기서 CLI는 draw.io 데스크탑 앱의 커맨드라인 익스포트 기능이다. 그러나 **devcontainer 환경에서 이 CLI를 동작시키려면** draw.io가 Electron 기반이라 `xvfb`와 대량의 X11 의존 패키지가 필요해서, Docker 이미지가 500MB 이상 불어난다. VS Code 확장은 `devcontainer.json`에 한 줄 추가하는 것으로 끝난다. **devcontainer 환경에서의 현실적인 답은 Skill + 확장이다.**

---

## **작업 흐름:**

```
AI에게 자연어로 지시
  → .drawio 파일 생성
  → VS Code에서 프리뷰 (Draw.io 확장이 자동으로 열림)
  → GUI로 색·배치·연결선 수동 미세조정
  → 확장 메뉴에서 PNG / SVG로 익스포트
  → git add → commit → push
```

지시 방법은 단순하다. `.drawio로 만들어줘`라고 말하면 AI가 Skill 정의를 참조해 `.drawio` 파일을 생성한다. 플로우차트뿐 아니라 아키텍처 도, 시퀀스 도도 자연어로 지시하면 된다.

AI가 **구조의 80%를 만들고, 사람이 GUI로 나머지 20% 외관을 다듬는** 하이브리드 방식이 핵심이다. Mermaid·PlantUML은 색을 바꾸려면 소스코드를 수정하고 재렌더링해야 하지만, Draw.io는 마우스로 직접 드래그해서 색을 바꾸면 끝이다.

---

## **GitHub에서의 관리 — `.drawio.svg` 이중 저장 전략:**
`.drawio` 파일은 그대로 Git 커밋 가능하다. 단, **GitHub는 `.drawio` 파일을 렌더링하지 않아** PR을 열어도 리뷰어는 생 XML만 보게 된다.

해결책은 **`.drawio.svg` 형식**이다. VS Code 확장에서 SVG로 익스포트하면 SVG 안에 Draw.io XML이 임베드된다. GitHub 위에서는 이미지로 표시되므로 PR에서 리뷰어가 도를 확인할 수 있다. 게다가 draw.io로 다시 열면 재편집도 가능하다.

- `.drawio` → 편집용 소스
- `.drawio.svg` → GitHub 표시·PR 리뷰용

**두 파일을 함께 커밋하는 것을 권장한다.** CI로 자동 변환하고 싶으면 `render-drawio-action`도 선택지에 들어간다.

---

## **MCP vs Skill + 확장 비교표:**

| 비교 항목 | MCP Tool Server | Skill + 확장 |
|---|---|---|
| 출력 | URL → 브라우저에서 확인 | `.drawio` 파일 → VS Code에서 확인 |
| Git 관리 | **불가** (파일이 남지 않음) | **가능** (`.drawio` 커밋) |
| 입력 포맷 | XML / Mermaid / CSV | XML만 |
| 컨텍스트 소비 | 3툴 분 정의가 항상 로드됨 | 필요할 때만 읽힘 (평소엔 0) |
| 편집 환경 | 브라우저 (app.diagrams.net) | VS Code 내부 |
| 오프라인 | 초회 다운로드 후 가능 | 완전 오프라인 대응 |
| 세팅 | `.mcp.json` + Node.js | `SKILL.md` 1개 + 확장 |

Mermaid 입력(토큰 효율 약 1/10)에 대응하는 것은 **MCP만**이다. Git 관리가 가능한 것은 **Skill + 확장만**이다.

---

## **어떤 툴을 선택할지 — 판단 기준:**

- **Git으로 관리하고 싶다** → Skill + 확장. 이것이 유일한 선택
- **Mermaid에서 변환하고 싶다** → MCP Tool Server
- **URL을 공유해서 상대방에게 바로 보여주고 싶다** → MCP Tool Server
- **VS Code 안에서 완결하고 싶다** → Skill + 확장
- **둘 다 쓰고 싶다** → 병용 가능. 같은 리포지터리의 별도 접근법이라 충돌하지 않는다

---

## **전체 도해 툴 사용 구분:**
Draw.io뿐만 아니라 Mermaid, PlantUML, HTML 기반 도해도 선택지다. 목적에 맞게 고르는 것이 중요하다.

| 유스케이스 | 추천 툴 | 이유 |
|---|---|---|
| 블로그 기사의 플로우 도·개념도 | HTML + Mermaid / PlantUML | PNG 직접 출력, 텍스트로 완결 |
| 텍스트로 관리할 사양서의 도 | Mermaid / PlantUML | diff가 깔끔, 텍스트로 완결 |
| 디자인에 공을 들이고 싶은 도 | Draw.io (Skill + 확장) | GUI 편집, 디자인 자유도 높음 |
| Mermaid / CSV에서 변환 | Draw.io MCP | 변환 대응은 이것뿐 |
| URL 붙여넣기로 즉시 공유 | Draw.io MCP | 링크 하나로 상대방이 볼 수 있음 |

Draw.io의 단점도 명확하다. `.drawio`의 내부는 XML이라 **Git diff가 보기 어렵고**, GitHub에서 렌더링되지 않는다는 점(`.drawio.svg`로 회피 가능)이 있다.

**판단 흐름을 한 줄로 정리하면:** 텍스트 관리·diff 중시 → Mermaid/PlantUML, GUI 편집·Git 관리 → Skill + 확장, GUI 편집·URL 공유 → MCP다.  