# Claude Code 실전 추천 Skills 정리
  
[출처](https://zenn.dev/imohuke/articles/claude-code-mcp-skills-summary )

  
## **Skills란?**
Claude Code의 기능을 확장하기 위한 **태스크 실행 절차 및 베스트 프랙티스를 담은 매뉴얼**이다. `skills.sh` 플랫폼에서 필요한 기능을 설치하면 Claude가 "할 수 있는 것"과 "알고 있는 것"을 비약적으로 늘릴 수 있다. 이것들을 도입하는 것만으로도 디자인, 코드 리뷰, DB 최적화, 마케팅 분석까지 혼자 완결시키는 "슈퍼 풀스택" 상태에 가까워진다.

---

## **🚀 필수 메타 스킬**
스킬을 탐색하고 만들기 위한 기반 스킬이다.

### **1. find-skills — "스킬을 위한 스킬"**
방대한 라이브러리 중에서 지금 하고 싶은 작업에 최적인 스킬을 자연어로 검색할 수 있다.

- **활용 예**: "이미지 최적화하는 스킬 있어?"라고 물으면 자동으로 제안해 준다.
- **출처**: [find-skills - Skills.sh](https://skills.sh/vercel-labs/skills/find-skills)

### **2. skill-creator — "나만의 무기를 만든다"**
독자적인 업무 플로우(예: 특정 포맷의 일일 보고서 작성, 독자 API 조작 등)를 자동화하는 스킬 자체를 생성한다.

- **활용 예**: 세금 신고 집계나 자체 서비스 관리 화면 조작을 자동화하는 스킬 작성.
- **출처**: [Skill Creation Process - Skills.sh](https://skills.sh/supabase/agent-skills/skill-creator)

---

## **🎨 디자인·프론트엔드 강화**
개인 개발에서 가장 시간이 소모되는 "디자인"과 "품질 보증"을 AI에게 맡긴다.

### **3. ui-ux-pro-max — "프로급 디자인을 복붙"**
참고 사이트 URL을 넘겨주기만 하면, 그 UI/UX(배색, 레이아웃, 폰트)를 분석해 자신의 프로젝트에서 사용할 수 있는 코드(Tailwind CSS 등)로 재현해 준다.

- **활용 예**: "이 대시보드 같은 느낌으로 설정 화면을 만들어줘"라고 지시.
- **출처**: [UI/UX Pro Max - Skills.sh](https://skills.sh/nextlevelbuilder/ui-ux-pro-max-skill/ui-ux-pro-max)

### **4. vercel-react-best-practices — "Vercel 엔지니어에 의한 자동 코드 리뷰"**
Vercel 공식 지식을 바탕으로 Next.js/React 코드를 진단한다. 불필요한 렌더링이나 번들 사이즈 비대화를 방지한다.

- **활용 예**: `use client`의 적절한 사용 위치 지적, 이미지 최적화 제안.
- **출처**: [vercel-react-best-practices - Skills.sh](https://skills.sh/supercent-io/skills-template/vercel-react-best-practices)

---

## **🗄️ 백엔드·인프라 (Supabase / Stripe)**
모던 개인 개발 스택에 필수적인 스킬이다.

### **5. supabase-postgres-best-practices — "DB 전문가를 고용하는 대신"**
Supabase 공식 스킬이다. 인덱스 누락, RLS(Row Level Security) 설정 실수 등 보안과 퍼포먼스에 관한 부분을 자동으로 감사한다.

- **활용 예**: "이 테이블 설계에 퍼포먼스 문제는 없어?"라고 리뷰 요청.
- **출처**: [Supabase Postgres Best Practices - Skills.sh](https://skills.sh/supabase/agent-skills/supabase-postgres-best-practices)

### **6. stripe-best-practices — "결제 구현 사고를 막는다"**
복잡해지기 쉬운 Stripe 구현(Webhook 처리, 구독 관리 등)에 대해 올바른 구현 패턴을 제안한다.

- **활용 예**: 과금 플로우 구현 시 공식 문서를 왔다갔다 하는 시간을 절감.
- **출처**: [stripe-best-practices - Skills.sh](https://skills.sh/exceptionless/exceptionless/stripe-best-practices)

---

## **📈 자동화·마케팅**
만든 것을 "운영"하고 "개선"하기 위한 스킬이다.

### **7. browser-use — "브라우저 조작 완전 자동화"**
AI가 브라우저를 열어 클릭이나 입력을 대신 수행한다. E2E 테스트나 경쟁사 조사에 강력한 효과를 발휘한다.

- **활용 예**: "내 사이트에 로그인해서 결제 화면까지 에러가 없는지 테스트해줘"라고 지시.
- **출처**: [Browser Automation - Skills.sh](https://skills.sh/browser-use/browser-use/browser-use)

### **8. funnel-analysis & cro-methodology — "매출을 늘리는 분석관"**
Google Analytics나 Mixpanel과 연계해 사용자가 어디서 이탈하는지 분석한다. 또한 심리학에 기반한 개선안(CRO, 전환율 최적화)도 제시한다.

- **활용 예**: LP 등록률이 낮은 원인을 특정하고 카피 수정안을 받는다.
- **출처**: [Mixpanel Automation - Skills.sh](https://skills.sh/composiohq/awesome-claude-skills/mixpanel-automation)

---

## **전체 요약**

| 카테고리 | 스킬 | 핵심 역할 |
|----------|------|-----------|
| 메타 | find-skills | 자연어로 스킬 검색 |
| 메타 | skill-creator | 나만의 커스텀 스킬 생성 |
| 프론트엔드 | ui-ux-pro-max | URL만으로 UI 재현 |
| 프론트엔드 | vercel-react-best-practices | Next.js/React 자동 코드 리뷰 |
| 백엔드 | supabase-postgres-best-practices | DB 보안·퍼포먼스 자동 감사 |
| 백엔드 | stripe-best-practices | 결제 구현 패턴 자동 제안 |
| 자동화 | browser-use | 브라우저 조작 자동화 |
| 마케팅 | funnel-analysis & cro-methodology | 이탈 분석 + CRO 개선안 제시 |

이 스킬들을 조합하면 **"기획 → 디자인 → 구현 → 테스트 → 분석"** 개발 사이클 전체를 혼자서 빠르게 돌릴 수 있다. 또한 필요에 따라 MCP를 조합하면 외부 서비스 연계도 자동화 가능하다.

**시작 방법:**

```bash
# 먼저 find-skills부터 설치
/add-skill find-skills
```   