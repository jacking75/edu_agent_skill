# Google Antigravity — Skills 기능 완전 정리

[출처](https://zenn.dev/emp_tech_blog/articles/google-antigravity-skills )   
   
---

## **개요:**
Google Antigravity에 새롭게 추가된 **Skills** 기능은, 에이전트에게 특정 태스크의 진행 방식과 베스트 프랙티스를 가르치는 **재사용 가능한 패키지**다. 단순한 프롬프트 템플릿이 아니라, 스크립트·체크리스트·예시 파일을 조합해 복잡한 작업을 자율적으로 수행시킬 수 있다.

---

## **Skills의 구성 요소 (4가지):**
Skills는 프로젝트의 `.agent/skills/` 디렉토리에 배치하며, 아래 4가지 요소로 구성된다.

- **`SKILL.md`**: 지시서 — 언제, 어떻게 동작할지를 정의한다
- **`scripts/`**: 도구 — 에이전트가 실행하는 Python 스크립트 등
- **`resources/`**: 소재 — 체크리스트, 템플릿, 사내 데이터 등
- **`examples/`**: 모범 답안 — 이상적인 출력 예시, 코드 규약 등

폴더 구조 자체가 에이전트에 대한 입력이 된다는 점이 특징이다.

---

## **Skills를 사용하는 3가지 이유:**

### **① 컨텍스트 절약과 "집중력" 유지**
Customizations에 모든 업무 매뉴얼을 넣으면 항상 대량의 토큰을 소비할 뿐만 아니라, 에이전트가 지시의 우선순위를 잃어(할루시네이션) 정확도가 떨어진다. Skills는 **필요할 때만 읽히는(Dynamic Loading)** 방식이므로, 에이전트가 눈앞의 태스크에 집중할 수 있다.

### **② "확률"과 "로직"의 장점만 취한 분업 체계**
LLM은 자연스러운 문장 생성은 잘하지만, 글자 수 계산이나 복잡한 연산 같은 엄밀한 처리는 취약하다. Skills를 사용하면 **엄밀한 체크는 `scripts/`(Python 등)에 맡기고, 결과를 인간답게 전달하는 부분은 LLM이 담당**하는 분업 구조를 구축할 수 있다.

### **③ Git 버전 관리 및 팀 공유**
Skills는 폴더와 파일의 실체로 존재하므로 Git으로 관리할 수 있다. 신규 멤버가 합류하면 `git pull` 한 번으로 팀 전체가 동일한 품질 기준(Lint 룰, 리뷰 관점)을 즉시 공유할 수 있다.

---

## **실전 예시: `code-reviewer` 스킬 구축:**
React 리포지토리를 대상으로 자동 코드 리뷰 스킬을 구축한다. 아래의 3단계 동작을 수행한다. 첫째, 코드를 받으면 스크립트로 기계적 실수(TODO 잔존 등)를 체크한다. 둘째, 체크리스트를 참조해 보안·성능 관점 누락을 방지한다. 셋째, 모범 답안에 따라 친절하고 건설적인 톤으로 지적한다.

### **폴더 구성:**

```
.
├── .agent/
│   └── skills/
│       └── code-reviewer/
│           ├── SKILL.md                  # 메인 지시서
│           ├── scripts/
│           │   └── simple_linter.py      # 간이 체크 툴
│           ├── resources/
│           │   └── review_checklist.md   # 리뷰 관점 리스트
│           └── examples/
│               └── review_style.md       # 피드백 작성 방식 견본
└── react/                                # 리뷰 대상 코드
```

### **① `SKILL.md` — 지시서**
YAML 프론트매터로 이름과 설명을 정의하고, 각 리소스를 연결한다.

```markdown
---
name: code-reviewer
description: 코드의 품질, 안전성, 가독성을 체크하고 개선안을 제시하는 스킬. PR 리뷰나 리팩토링 상담 시 사용.
---

# Code Reviewer Skill

## 리뷰 절차

1. **자동 체크 실행**
   - 대상 코드 파일이 있는 경우: `python scripts/simple_linter.py <파일경로>` 실행
   - 코드가 채팅에 직접 붙여넣기된 경우: 이 단계 스킵

2. **관점 확인**
   - `resources/review_checklist.md` 를 읽어 각 항목(보안, 성능 등) 확인

3. **피드백 작성**
   - `examples/review_style.md` 의 톤 & 매너 참조
   - 출력 구성: 전체 감상 → 중요 과제 → 개선 제안 → 수정 코드 예시

## 주의 사항
- 비판만 하지 말고, 왜 수정이 필요한지(Why)를 설명한다
- 변수명·주석 부족도 적극적으로 지적한다
```

### **② `scripts/simple_linter.py` — 도구**
LLM만으로 리뷰하면 놓치기 쉬운 단순한 문자열 매칭을 스크립트에 맡긴다.

```python
import sys
import re

def analyze_code(file_path):
    patterns = {
        "FIXME/TODO": r"(TODO|FIXME)",
        "Print Statement": r"(print\(|console\.log\()",
        "Hardcoded Token": r"(token|password|secret)\s*=",
        "Generic Exception": r"except Exception:",
    }

    print(f"--- Analyzing {file_path} ---")
    issues_found = False

    try:
        with open(file_path, 'r', encoding='utf-8') as f:
            lines = f.readlines()

        for i, line in enumerate(lines):
            for issue_name, pattern in patterns.items():
                if re.search(pattern, line):
                    level = "WARN" if "Print" in issue_name else "review-required"
                    print(f"Line {i+1} [{level}]: found {issue_name} -> {line.strip()}")
                    issues_found = True

        if not issues_found:
            print("No obvious issues found by simple_linter.")

    except FileNotFoundError:
        print("Error: File not found.")

if __name__ == "__main__":
    if len(sys.argv) < 2:
        print("Usage: python simple_linter.py <file_path>")
    else:
        analyze_code(sys.argv[1])
```

### **③ `resources/review_checklist.md` — 소재**
리뷰의 "질"을 보장하는 체크 항목이다. 사내 보안 기준 등을 여기에 기재하면 강력해진다.

```markdown
# 코드 리뷰 체크리스트

## 🛡️ 안전성 (Security)
- SQL 인젝션, XSS 취약점은 없는가?
- 패스워드·API 키가 하드코딩되어 있지 않은가?
- 입력값 검증(Validation)은 적절한가?

## 🚀 성능 (Performance)
- 루프 안에서 무거운 처리(DB 접속 등)를 하지 않는가?
- 불필요한 메모리 할당은 없는가?
- N+1 문제가 발생할 가능성은 없는가?

## 📖 가독성·유지보수성 (Readability)
- 변수명·함수명이 "무엇을 하는지"를 표현하고 있는가? (`data`, `temp` 같은 모호한 이름은 NG)
- 복잡한 로직에 주석이 달려 있는가?
- 함수가 너무 길지 않은가? (단일 책임 원칙)
- 에러 핸들링이 적절한가? (에러를 묵살하지 않는가?)

## 🧪 테스트 (Testing)
- 엣지 케이스(경계값, null, 빈 리스트 등)는 고려되었는가?
```

### **④ `examples/review_style.md` — 모범 답안**
에이전트의 "말투"와 "지적 포맷"을 제어한다. 이것만 있어도 "AI스러운 답변"이 "시니어 엔지니어스러운 답변"으로 바뀐다.

````markdown
# 좋은 리뷰 작성 예시

## 나쁜 예 ❌
> 10번째 줄의 변수명이 알아보기 어렵습니다. 고쳐주세요. 루프도 비효율적입니다.

## 좋은 예 ✅

### 전체 감상
기능 구현 감사합니다! 로직은 명확하고 잘 동작하는 것 같습니다.
유지보수성과 성능 관점에서 몇 가지 제안이 있습니다.

### 개선 제안

**1. 변수명 구체화 (Line 10)**
`d`라는 변수명은 나중에 보는 사람이 어떤 데이터인지 알기 어렵습니다.
날짜 데이터라면 `target_date`, 딕셔너리라면 `user_profile` 등으로 변경을 권장합니다.

**2. 루프 내 계산 (Line 15-20)**
루프 안에서 매번 `calculate_tax_rate()`를 호출하고 있으나, 세율은 고정값이므로
루프 밖으로 꺼내면 성능이 향상됩니다.

```python
# 수정안
tax_rate = calculate_tax_rate()  # 루프 밖으로 이동
for item in items:
    price = item.price * tax_rate
```

**3. 에러 핸들링**
`except Exception:`으로 모든 에러를 캐치하면 예상치 못한 버그를 놓칠 수 있습니다.
`ValueError` 등 예상되는 에러만 캐치하도록 변경하는 것이 좋습니다.
````

---

## **실제 동작 결과:**
"1개 전 변경사항을 코드 리뷰해줘"라고만 입력했을 때, 에이전트는 자동으로 `code-reviewer` 스킬을 인식하고 발동시켰다. "스킬을 사용해"라는 명시적 지시 없이도 스크립트 실행 → 체크리스트 참조 → 스타일 가이드에 맞는 피드백 생성까지 자율적으로 수행했다.

---

## **다른 활용 아이디어:**

### **① 테스트 코드 생성 (`test-generator`)**
`examples/`에 자사 프로젝트의 테스트 작성 방식(pytest fixture 사용법 등)을 배치해 두면, 누가 작성해도(AI가 작성해도) 동일한 작법의 테스트 코드가 생성된다.

### **② SQL 쿼리 작성 (`sql-expert`)**
`resources/`에 `schema.md`(테이블 정의서)를 배치해 두면, "지난달 매출 뽑아줘"라고 말하는 것만으로 올바른 테이블 조인이 포함된 SQL이 출력된다. 매번 스키마를 설명할 필요가 없어진다.

### **③ 버그 보고서 작성 (`bug-reporter`)**
`resources/`에 버그 보고 Markdown 템플릿을, `scripts/`에 로그 수집 스크립트를 배치해 두면, 에러 발생 시 "보고서 만들어줘" 한 마디로 로그가 포함된 깔끔한 티켓 문서가 완성된다.

---

## **핵심 요약:**

| 요소 | 역할 |
|---|---|
| `SKILL.md` | 에이전트 지시서, 동작 트리거 및 절차 정의 |
| `scripts/` | 기계적·엄밀한 처리를 담당 (LLM의 약점 보완) |
| `resources/` | 체크리스트, 스키마 등 판단 기준 소재 |
| `examples/` | 출력 포맷·말투 제어, "AI스러움" 탈피 |

Skills의 핵심 가치는 세 가지다. 필요할 때만 동작하는 Dynamic Loading으로 정확도를 높이고, LLM과 스크립트의 분업으로 강점만 살리며, Git 관리로 팀 전체가 동일한 품질 기준을 즉시 공유할 수 있다.  