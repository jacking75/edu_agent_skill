# Claude Code → GitHub Copilot Agent Skills 이식 시 발화 안 되는 문제 완전 정리
  
[출처](https://tech-lab.sios.jp/archives/51159 )

> **검증 환경**: 2026년 2월 2일 / VS Code 1.108 이상 + GitHub Copilot

---

## **개요**
Claude Code에서 사용하던 Skills(SKILL.md 형식)를 GitHub Copilot으로 그대로 이식했을 때, **스킬이 전혀 발화(trigger)되지 않는 문제**가 발생한다. 같은 SKILL.md 형식임에도 동작하지 않는 근본 원인과 3가지 해결책을 정리한다.

---

## **발화 안 되는 근본 원인: 발화 메커니즘의 차이**
Claude Code와 GitHub Copilot은 스킬 발화 메커니즘이 **근본적으로 다르다**.

| 관점 | Claude Code | GitHub Copilot |
|------|------------|----------------|
| 스킬 파악 방식 | 시작 시 **전체 스킬 메타데이터를 모두 파악** | relevance(관련도) 기반으로 선택적 로딩 |
| 판단 방식 | 전체 정보 파악 후 내부 판단 (정보 완전) | **확률적 판정 (사전 필터링 있음)** |
| description 역할 | 발화 판정에 사용 (전 메타데이터 파악 후 판단) | **발화 판정에 사용 (사전 필터링 단계에서 판단)** |

두 시스템 모두 **Progressive Disclosure** 설계 사상을 채용하고 있다. 전체 스킬을 미리 로딩하지 않고, 관련도에 따라 필요한 스킬만 로딩하는 구조다. 그러나 **"어느 단계에서 좁히는가"** 에 차이가 있다.

- **Claude Code**: 전체 스킬을 파악한 뒤, 내부 판단으로 선택 → description이 다소 허술해도 의도를 이해
- **GitHub Copilot**: 사전에 먼저 필터링한 후 선택 → description 작성 방식에 따라 "누락"이 발생

즉, **Claude Code 감각으로 대충 만들면 Copilot에서는 동작하지 않는다**.

---

## **핵심 차이: description 설계 사상**

**Claude Code**에서는 "무엇을 하는가"만 써도 충분히 동작한다.

```yaml
description: 블로그 기사를 스크레이핑해서 Markdown으로 변환
```

내부 라우팅이 뛰어나기 때문에 이 정도 설명으로도 의도를 파악한다.

**GitHub Copilot**에서는 "스킬의 기능 + 발화 조건" **양쪽 모두**를 써야 한다. 공식 가이드라인에서 "describe BOTH what the skill does AND when to use it"이라고 명시하고 있다. Copilot의 관련도 스코어링은 **사용자가 실제로 사용하는 단어**를 description에 넣지 않으면 스코어가 오르지 않는다.

| 항목 | Claude Code | GitHub Copilot |
|------|------------|----------------|
| 설정의 엄밀함 | 대충 작성해도 동작 | **엄밀하게 작성 필요** |
| 토큰 예산 | 신경 쓰지 않아도 됨 | **500토큰 이하 권장** (상한 5,000토큰) |
| 결과 | – | **작법에 따르면 동등한 기능 구현 가능** |

---

## **해결책 ① — description 강화**

**주의: "영어로 써야 한다"는 팁이 역효과가 되는 경우가 있다.**

Copilot의 관련도 판정은 **문자열 유사도**로 판단한다고 추정된다. 사용자 발화와 description의 단어가 가까울수록 스코어가 높아진다. 한국어로 프롬프트를 입력하는 사용자에게는 **해당 언어의 발화 키워드**가 필요하다. 영어로만 쓰면 한국어 쿼리와의 유사도가 낮아져 발화하지 않게 된다. **결론: description에는 한국어·영어 양쪽 발화 키워드를 모두 열거한다.**

**권장 포맷:**

```yaml
description: |
  [기능의 간결한 설명 (어떤 스킬인가)].
  Use when: [어떤 때 사용하는가 (사용자의 의도)].
  Triggers on: [발화 키워드 (한국어·영어 모두)].
```

**Before (발화 안 됨):**

```yaml
description: HTML을 Playwright로 렌더링해서 PNG 이미지로 변환
```

**After (발화됨):**

```yaml
description: |
  HTML 파일을 Playwright로 렌더링해서 PNG 이미지로 변환하는 스킬.
  Use when: HTML을 이미지로 만들고 싶다, 스크린샷을 찍고 싶다.
  Triggers on: HTML을 PNG로 변환, HTML to PNG, 이미지로 변환, screenshot HTML, HTML 캡처.
```

포인트는 "기술적으로 무엇을 하는가"가 아닌, **"사용자가 어떻게 말하는가"**를 상상해서 작성하는 것이다.

---

## **해결책 ② — 토큰 예산 준수**
Copilot은 토큰 효율을 중시하는 설계다. SKILL.md가 너무 길면 아예 읽히지 않을 가능성이 있다.

| 파일 | 권장 | 상한 |
|------|------|------|
| SKILL.md | **500토큰 이하** (약 60줄 기준) | 5,000토큰 |
| references/*.md | 1,000토큰 | 2,000토큰 |

길어질 것 같은 경우는 세부 내용을 `references/`로 분리한다.

```
.github/skills/my-skill/
├── SKILL.md               # 개요 + Quick Start (~60줄)
└── references/
    ├── commands.md        # 커맨드 상세
    └── troubleshooting.md
```

SKILL.md는 "라우터 역할"에 집중하고, 상세 내용은 필요할 때만 로딩되도록 구성한다.

---

## **해결책 ③ — Instructions 라우팅 (강제 방법)**
관련도 스코어링은 불안정할 수 있다. description을 강화해도 발화하거나 하지 않는 경우가 생긴다. 그 때의 최후 수단으로, **`copilot-instructions.md`에 명시적인 라우팅 테이블을 추가**하는 방법이 있다.

```markdown
## Agent Skills Routing

| 키워드                          | 스킬                  | 경로                                        |
|---------------------------------|-----------------------|---------------------------------------------|
| 블로그 스크레이핑, tech-lab     | blog-scraper          | `.github/skills/blog-scraper/SKILL.md`      |
| 플로우차트, 다이어그램          | html-diagram          | `.github/skills/html-diagram/SKILL.md`      |
```

**왜 효과적인가:**
- Instructions는 **항상 읽힌다**
- 관련도 스코어링을 **우회**할 수 있다
- 발화가 **안정화**된다

**단점(트레이드오프):**
- **유지보수성 저하**: 스킬을 추가·변경할 때마다 라우팅 테이블 업데이트 필요
- **확장성**: 스킬 수가 늘어나면 Instructions 파일이 비대해짐
- **본래 설계 사상에서 벗어남**: Progressive Disclosure의 혜택을 받지 못함

**결론**: 반드시 발화시켜야 하는 중요한 스킬 (3~5개 정도)에 한정해서 사용하는 것을 권장한다.

**실제 검증 결과 (Claude Code에서 이식한 4개 스킬):**

| 스킬 | 프롬프트 | 결과 | 발화 경로 |
|------|---------|------|-----------|
| blog-scraper | `tech-lab.sios.jp + 가져와` | ✅ | Instructions |
| html-diagram | `HTML로 도식화해` | ✅ | Instructions |
| copilot-chat-converter | `Markdown으로 변환해` | ✅ | Instructions |
| html-to-png | 연계 호출 | ✅ | 스킬 내부 |

Instructions 라우팅 테이블로 전체 스킬이 안정적으로 발화됐다.

---

## **이식 체크리스트**
Claude Code Skills → GitHub Copilot 이식 절차다.

```
- [ ] `.claude/skills/` → `.github/skills/` 로 복사
- [ ] `allowed-tools` 삭제 (Copilot에서는 실험적 기능)
- [ ] description에 `Use when` / `Triggers on` 추가
- [ ] 사용자 발화를 한국어·영어 모두 열거
- [ ] SKILL.md를 500토큰 이하 (약 60줄 기준)로 축소
- [ ] `copilot-instructions.md`에 라우팅 테이블 추가 (중요 스킬만)
- [ ] VS Code 리로드 후 5~10분 대기
- [ ] 발화 테스트 실시
```

---

## **전체 요약**

| 포인트 | 내용 |
|--------|------|
| 원인 | Claude Code(전체 스킬 파악)와 GitHub Copilot(사전 필터링)의 발화 구조가 다르다 |
| description | "무엇을 하는가" + **"언제 사용하는가 (Use when / Triggers on)"** 양쪽 모두 작성 |
| 해결책 | ① description 강화 → ② 토큰 예산 준수 → ③ **Instructions 라우팅 (강제 방법)** |
| 결론 | **작법에 따라 엄밀하게 만들면, Claude Code와 동등한 기능 구현 가능** |

발화하지 않을 경우, 위 3가지 해결책을 순서대로 시도한다. 특히 Instructions 라우팅은 확실성이 높아 가장 효과적이다.  