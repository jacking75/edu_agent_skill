# 🧠 Memvid — AI 에이전트를 위한 단일 파일 메모리 레이어

## **개요 (What is Memvid?)**
Memvid는 AI 에이전트에게 **지속적이고 이식 가능한 장기 기억(Long-term Memory)** 을 제공하기 위해 설계된 오픈소스 메모리 시스템입니다. 핵심 아이디어는 단순하지만 강력합니다. 데이터, 임베딩(Embeddings), 검색 인덱스, 메타데이터 — 이 모든 것을 단 하나의 `.mv2` 파일 안에 패키징하는 것입니다. 마치 SQLite가 관계형 데이터베이스 전체를 하나의 파일로 만든 것처럼, Memvid는 AI 메모리 시스템 전체를 하나의 파일로 만들어버립니다.

프로젝트는 2025년 5월 Python 기반의 실험적 v1으로 처음 공개되었고, 2026년 1월에는 **Rust로 완전 재작성된 v2(v2.0.131)** 가 출시되며 10~100배의 성능 향상을 이루어냈습니다. Apache-2.0 라이선스로 배포되는 완전한 오픈소스입니다.

---

## **탄생 배경 — 해결하려는 문제**
AI 에이전트 개발에서 가장 골치 아픈 문제 중 하나는 **상태 비저장성(Statelessness)** 입니다. GPT-4나 Claude 같은 대형 언어 모델들은 대화가 끝나면 모든 것을 잊어버립니다. 이를 해결하기 위한 표준 솔루션인 RAG(Retrieval-Augmented Generation)는 효과적이지만, 그 대가로 엄청난 인프라 부담을 요구합니다. Pinecone, Weaviate, ChromaDB 같은 벡터 데이터베이스를 운용하려면 별도 서버 프로세스 실행, 클라우드 API 키 관리, 네트워크 레이턴시 부담, 클라우드 비용, 그리고 복잡한 디버깅 등이 뒤따릅니다.

Memvid의 창시자는 개인 문서 컬렉션을 인덱싱하는 과정에서 전통적인 벡터 데이터베이스의 높은 RAM 소비와 클라우드 비용에 좌절하며 이 프로젝트를 시작했습니다. 그 결과물이 **"인프라 세금(Infrastructure Tax) 없는 AI 메모리"** 라는 개념의 Memvid입니다.

---

## **핵심 아키텍처**

**Smart Frame — 비디오 인코딩에서 영감을 받은 메모리 단위**

Memvid의 가장 독창적인 설계 개념은 'Smart Frame'입니다. 이는 비디오 인코딩 방식에서 영감을 받은 것으로, 실제 비디오를 저장하는 것이 아니라 메모리를 **비디오의 프레임처럼 순차적이고 불변(Immutable)한 단위로 조직화** 합니다. 각 Smart Frame은 내용물, 타임스탬프, 체크섬, 기본 메타데이터를 저장하는 독립적인 단위입니다. 프레임은 효율적인 압축, 인덱싱, 병렬 읽기를 허용하는 방식으로 그룹화됩니다.

이 프레임 기반 설계의 핵심적인 특성은 **Append-Only 쓰기** 방식입니다. 새로운 메모리가 추가될 때 기존 데이터를 수정하거나 덮어쓰지 않고 새 프레임을 뒤에 붙이는 방식으로 동작합니다. 이 덕분에 특정 시점의 메모리 상태로 되감기(Time-Travel)하거나, 지식이 어떻게 진화해왔는지 타임라인으로 검사하는 것이 가능해집니다. 또한 충돌(Crash) 발생 시에도 커밋된 불변 프레임들은 안전하게 보존됩니다.

**`.mv2` 파일 포맷 — 모든 것이 하나의 파일 안에**

`.mv2` 파일은 내부적으로 다음과 같은 구조를 가집니다.

```
┌─────────────────────────────────────┐
│  Header (4KB)                       │  매직 넘버, 버전, 용량 정보
├─────────────────────────────────────┤
│  Embedded WAL (1-64MB)              │  충돌 복구용 Write-Ahead Log
├─────────────────────────────────────┤
│  Data Segments                      │  압축된 Smart Frames (실제 데이터)
├─────────────────────────────────────┤
│  Lex Index                          │  BM25 기반 전문 검색 인덱스 (Tantivy)
├─────────────────────────────────────┤
│  Vec Index                          │  HNSW 벡터 유사도 검색 인덱스
├─────────────────────────────────────┤
│  Time Index                         │  시간순 정렬 인덱스
├─────────────────────────────────────┤
│  TOC (Footer)                       │  각 세그먼트의 오프셋 정보
└─────────────────────────────────────┘
```

중요한 점은 `.wal`, `.lock`, `.shm`, 사이드카 파일 같은 **보조 파일이 전혀 생성되지 않는다**는 것입니다. 파일은 단 하나이며, 그것이 전부입니다.

**하이브리드 검색 엔진**

Memvid는 두 가지 검색 방식을 동시에 지원합니다. 첫 번째는 **BM25 렉시컬 검색(Lexical Search)** 으로, Tantivy 라이브러리 기반의 전문 검색(Full-text Search)이며 키워드 정확도 매칭에 최적화되어 있습니다. 두 번째는 **벡터 시맨틱 검색(Semantic Search)** 으로, HNSW(Hierarchical Navigable Small World) 알고리즘과 로컬 ONNX 텍스트 임베딩 모델을 활용하여 의미론적 유사성을 기반으로 검색합니다. 기본 모드는 이 둘을 결합한 **하이브리드 검색**으로, 키워드 정확도와 의미론적 이해를 동시에 활용합니다.

**SlotIndex와 Memory Cards**

엔티티 기반 쿼리를 위한 별도의 SlotIndex가 존재합니다. 이는 Subject-Predicate-Object(주어-서술어-목적어) 삼중항(SPO Triplet) 형태로 구조화된 사실(Fact)을 저장하고, O(1) 상수 시간에 특정 엔티티의 상태를 조회할 수 있게 해줍니다. 예를 들어 "Alice"라는 엔티티를 조회하면 `{employer: 'Anthropic', role: 'Senior Engineer'}` 같은 구조화된 답변이 즉시 반환됩니다.

---

## **성능 벤치마크**
공식 GitHub에서 공개된 벤치마크 결과는 상당히 인상적입니다.

- **정확도**: LoCoMo 벤치마크 대비 SOTA(State-of-the-Art) 대비 **+35% 향상**
- **멀티-홉 추론(Multi-hop Reasoning)**: 업계 평균 대비 **+76% 향상**
- **시간적 추론(Temporal Reasoning)**: 업계 평균 대비 **+56% 향상**
- **레이턴시**: P50 기준 **0.025ms**, P99 기준 **0.075ms** 의 초저지연
- **처리량(Throughput)**: 표준 대비 **1,372배** 높은 처리량

---

## **장점**
Memvid의 가장 큰 강점은 **단순성과 이식성**입니다. `.mv2` 파일 하나를 복사하거나 Git에 커밋하거나 USB에 담는 것만으로 AI 에이전트의 전체 메모리가 이동합니다. 서버 설정, 환경 변수 관리, 네트워크 설정이 전혀 필요 없습니다. 인터넷 연결이 없는 환경(오프라인, 에어갭 네트워크)에서도 완벽하게 동작합니다.

**Time-Travel Debugging**은 기존 벡터 데이터베이스에서는 찾기 어려운 독보적인 기능입니다. 에이전트가 특정 시점에 무엇을 알고 있었는지를 정확히 재현할 수 있어, 에이전트가 왜 잘못된 답을 했는지 분석하고, 과거 특정 시점의 메모리 상태로 브랜치(branch)를 만들어 A/B 테스트를 수행하는 것이 가능합니다. 마치 Git의 브랜치 개념을 AI 에모리에 적용한 것과 같습니다.

또한 Embedded WAL(Write-Ahead Log)이 파일 내부에 내장되어 있어 갑작스러운 프로세스 종료나 전원 차단이 발생해도 데이터 손상 없이 자동 복구가 가능합니다. 비용 측면에서도 서버 기반 솔루션 대비 **최대 93% 비용 절감**이 가능하며, 모델에 종속되지 않는(Model-Agnostic) 설계로 OpenAI, Anthropic, Gemini 등 어떤 LLM과도 조합이 가능합니다.

---

## **단점 및 한계**
규모의 한계가 가장 명확한 제약 조건입니다. Memvid는 단일 파일 시스템이기 때문에 수백만 명의 동시 사용자를 처리해야 하는 대규모 멀티테넌트 SaaS 서비스에는 적합하지 않습니다. 파일 크기가 커질수록 성능에 영향을 줄 수 있으며, 분산 처리나 수평 확장(Horizontal Scaling)이 필요한 엔터프라이즈 레벨의 프로덕션 환경에서는 전통적인 벡터 데이터베이스가 더 적합합니다.

프로젝트의 역사가 짧다는 점도 리스크 요소입니다. v1(2025년 5월)에서 v2(2026년 1월)로 완전 재작성되는 과정에서 파일 포맷과 API가 크게 변경되었고, v1은 현재 공식적으로 Deprecated(폐기) 상태입니다. 아직도 활발히 개발 중인 프로젝트인 만큼, 미래의 파일 포맷 변경 가능성을 완전히 배제할 수 없습니다.

---

## **사용 방법**

### **설치**

```bash
# CLI 설치 (Node.js)
npm install -g memvid-cli

# Python SDK
pip install memvid-sdk

# Node.js SDK
npm install @memvid/sdk

# Rust (Cargo.toml에 추가)
# memvid-core = { version = "2.0", features = ["lex", "vec", "temporal_track"] }
```

### **CLI 기본 사용법**

```bash
# 새 메모리 파일 생성
memvid create knowledge.mv2

# 데이터 추가
echo "Alice works at Anthropic as a Senior Engineer." | memvid put knowledge.mv2

# 하이브리드 검색 (기본)
memvid find knowledge.mv2 --query "who works at AI companies"

# 렉시컬 검색만 사용
memvid find knowledge.mv2 --query "budget" --mode lex

# 시맨틱 검색만 사용
memvid find knowledge.mv2 --query "financial outlook" --mode sem

# 엔티티 상태 조회 (O(1))
memvid state knowledge.mv2 "Alice"
# { employer: 'Anthropic', role: 'Senior Engineer' }

# 사실 추출로 Memory Cards 생성
memvid enrich knowledge.mv2 --engine rules

# LLM 기반 자연어 질의응답
export OPENAI_API_KEY=sk-...
memvid ask knowledge.mv2 --question "What is Alice's role?" --use-model openai
```

### **Python SDK 사용법**

```python
from memvid_sdk import create, use
import os

# 파일이 있으면 열고, 없으면 새로 생성
path = 'knowledge.mv2'
mem = use('basic', path) if os.path.exists(path) else create(path)

# 데이터 추가
mem.put(title='Team Info', label='notes', metadata={}, text='Alice works at Anthropic as a Senior Engineer.')

# 하이브리드 검색
results = mem.find('who works at AI companies', k=5, mode='hybrid')

# 엔티티 상태 조회
alice = mem.state('Alice')
print(alice)  # {'slots': {'employer': 'Anthropic', 'role': 'Senior Engineer'}}
```

### **Node.js SDK 사용법**

```javascript
import { create, use } from '@memvid/sdk';
import { existsSync } from 'fs';

// 파일 존재 여부 확인 후 분기
const mem = existsSync('knowledge.mv2')
  ? await use('basic', 'knowledge.mv2')
  : await create('knowledge.mv2', 'basic');

// 데이터 추가
await mem.put({ title: 'Team Info', label: 'notes', text: 'Alice works at Anthropic...' });

// 검색
const results = await mem.find('who works at AI companies', { k: 5, mode: 'lex' });

// 엔티티 조회
const alice = await mem.state('Alice');
console.log(alice); // { slots: { employer: 'Anthropic', role: 'Senior Engineer' } }
```

### **Rust 사용법**

```rust
use memvid_core::{Memvid, PutOptions, SearchRequest};

fn main() -> memvid_core::Result<()> {
    // 새 메모리 파일 생성
    let mut mem = Memvid::create("knowledge.mv2")?;

    // 메타데이터와 함께 데이터 추가
    let opts = PutOptions::builder()
        .title("Meeting Notes")
        .uri("mv2://meetings/2024-01-15")
        .tag("project", "alpha")
        .build();
    mem.put_bytes_with_options(b"Q4 planning discussion...", opts)?;
    mem.commit()?;

    // 검색
    let response = mem.search(SearchRequest {
        query: "planning".into(),
        top_k: 10,
        snippet_chars: 200,
        ..Default::default()
    })?;

    for hit in response.hits {
        println!("{}: {}", hit.title.unwrap_or_default(), hit.text);
    }

    Ok(())
}
```

### **Time-Travel 디버깅**

```bash
# 세션 시작 및 기록
memvid session start knowledge.mv2 --name "qa-test"
memvid find knowledge.mv2 --query "test query"
memvid session end knowledge.mv2

# 다른 파라미터로 세션 재현
memvid session replay knowledge.mv2 --session abc123 --top-k 10
```

---

## **사용 시나리오**

**개인 AI 어시스턴트 개발**은 Memvid가 가장 빛나는 영역입니다. 사용자의 선호도, 이전 대화 내용, 개인화된 지식을 `.mv2` 파일에 지속적으로 누적하면, 사용자가 새 기기로 이동하거나 앱을 재설치할 때 파일 하나만 복사하면 모든 개인화된 기억이 그대로 이전됩니다.

**팀 코딩 표준 학습 에이전트**도 훌륭한 시나리오입니다. 코드 리뷰 에이전트가 PR 피드백을 반복해서 받으면서 팀의 코딩 컨벤션을 학습하고, 학습된 지식 전체를 하나의 `.mv2` 파일로 모노레포(monorepo)에 커밋하면, 새로운 팀원은 저장소를 클론하는 것만으로 팀의 기준을 알고 있는 AI 에이전트를 바로 사용할 수 있게 됩니다.

**오프라인/에어갭 환경 RAG 시스템** 구축에도 탁월합니다. 인터넷 연결이 제한된 의료, 법무, 금융 환경에서 내부 문서들을 `.mv2` 파일로 패키징하여 완전히 로컬에서 동작하는 지식 검색 시스템을 구축할 수 있습니다. 연구자들은 50,000편의 논문을 1시간 이내에 인덱싱하여 오프라인에서도 고속 시맨틱 검색이 가능한 연구 데이터베이스를 만들 수 있습니다.

**멀티-에이전트 시스템**에서는 여러 에이전트가 동일한 `.mv2` 파일을 공유 메모리로 사용하거나, 각 에이전트가 자신만의 메모리 파일을 가지고 필요할 때 병합하는 패턴도 가능합니다. LangChain, AutoGen, CrewAI 같은 주요 에이전트 프레임워크와의 통합도 공식 지원됩니다.

---

## **경쟁 솔루션 비교**

| 기능 | **Memvid** | Pinecone | ChromaDB | Weaviate |
|------|-----------|---------|---------|---------|
| **배포 형태** | 단일 `.mv2` 파일 | 클라우드 전용 | SQLite + 보조 파일 | Docker 컨테이너 |
| **오프라인 지원** | ✅ 완전 지원 | ❌ 제한적 | ✅ 지원 | ⚠️ 초기 설정 후 가능 |
| **하이브리드 검색** | ✅ BM25 + 벡터 | ✅ 지원 | ❌ 벡터만 | ✅ 지원 |
| **엔티티 O(1) 조회** | ✅ SlotIndex | ❌ 없음 | ❌ 없음 | ❌ 없음 |
| **Time-Travel 디버깅** | ✅ 내장 | ❌ 없음 | ❌ 없음 | ❌ 없음 |
| **충돌 안전성** | ✅ 내장 WAL | ☁️ 클라우드 처리 | ✅ SQLite 기반 | ☁️ 클라우드/컨테이너 |
| **대규모 멀티유저** | ❌ 제한적 | ✅ 최적화 | ⚠️ 제한적 | ✅ 지원 |
| **비용** | 무료 (로컬) | 유료 (클라우드) | 무료 | 무료/유료 |

---

## **요약**

Memvid는 AI 에이전트 메모리 분야에서 "SQLite의 철학"을 실현한 프로젝트입니다. 복잡한 인프라를 단 하나의 파일로 대체함으로써, 특히 개인 에이전트, 데스크톱 애플리케이션, 연구 프로토타입, 오프라인 환경, 소규모 팀 내부 도구 등 **전체 에이전트 사용 사례의 약 80%** 를 커버하는 간단하고 강력한 솔루션입니다. 수백만 명의 동시 접속이 필요한 대규모 SaaS 서비스가 아니라면, Memvid는 기존 벡터 데이터베이스를 완전히 대체할 수 있는 매력적인 선택지가 됩니다.  
  

---

## OpenClaw에서 Memvid 사용 가능한가?
**현재(2026년 2월 기준) Memvid를 OpenClaw에 직접 연동하는 공식 플러그인이나 ClawHub 등록 스킬은 존재하지 않습니다.** 그러나 OpenClaw의 구조를 이해하면 **직접 구현하는 것은 충분히 가능**합니다.

---

### **OpenClaw의 메모리 현황**
OpenClaw 자체는 세션이 끊기면 기억을 잃는 **상태 비저장(stateless)** 구조입니다. 이 문제를 해결하기 위해 현재 커뮤니티에서 활발히 사용되는 공식/비공식 메모리 솔루션들은 다음과 같습니다.

- **Mem0** — `@mem0/openclaw-mem0` 공식 플러그인이 존재하며, 가장 많이 사용되는 클라우드 기반 장기 메모리 솔루션입니다.
- **Supermemory** — OpenClaw 전용 플러그인이 존재하는 또 다른 클라우드 메모리 옵션입니다.
- **Cognee** — 로컬 Docker로 실행하는 그래프 기반 메모리 엔진으로, OpenClaw 연동 가이드가 있습니다.
- **QMD / Obsidian** — 파일 기반 메모리 접근법으로 커뮤니티에서 사용됩니다.
- **SOUL.md / MEMORY.md** — OpenClaw 내장 파일 기반 단순 메모리 방식입니다.

즉, **Memvid를 OpenClaw에 연결하는 공식 경로는 아직 없습니다.**

---

### **그럼에도 연동이 가능한 이유 — OpenClaw Skills 시스템**
OpenClaw는 `SKILL.md` 파일 하나로 정의되는 **Skills 시스템**을 통해 에이전트에게 새로운 능력을 부여할 수 있습니다. Memvid는 Python SDK(`memvid-sdk`), Node.js SDK(`@memvid/sdk`), CLI(`memvid-cli`) 를 모두 지원하므로, OpenClaw의 Skills 형식에 맞춰 직접 스킬을 작성하면 연동이 가능합니다.

**직접 구현 예시 — `SKILL.md` 작성**

```markdown
---
name: memvid-memory
description: >
  Persist and retrieve long-term memory using a local .mv2 file via Memvid.
  Use 'memvid_save' to store facts and 'memvid_search' to recall them.
metadata:
  {
    "openclaw": {
      "emoji": "🧠",
      "requires": { "bins": ["memvid"] },
      "install": [
        {
          "id": "node",
          "kind": "node",
          "package": "memvid-cli",
          "bins": ["memvid"],
          "label": "Install Memvid CLI"
        }
      ]
    }
  }
---

## Memvid Memory Skill

You have access to a persistent, local memory file at `~/.openclaw/memory.mv2`.

### When to save
- User explicitly asks you to remember something
- Important facts, preferences, or decisions are established
- After completing a significant task or project

### How to save
Run the following shell command:
```bash
echo "CONTENT_TO_REMEMBER" | memvid put ~/.openclaw/memory.mv2
```

### When to search
- At the start of any new conversation, always search for relevant context
- Before answering questions that might have historical context

### How to search
```bash
memvid find ~/.openclaw/memory.mv2 --query "SEARCH_QUERY"
```
```
```  

이 `SKILL.md` 파일을 `~/.openclaw/workspace/skills/memvid-memory/` 디렉터리에 넣으면 OpenClaw가 자동으로 로드합니다.

---

### **Memvid vs 현재 OpenClaw 메모리 솔루션 비교**

| 항목 | **Memvid (수동 연동)** | Mem0 (공식 플러그인) | Cognee (로컬) |
|---|---|---|---|
| **공식 지원** | ❌ 없음 (직접 구현) | ✅ 공식 플러그인 | ⚠️ 가이드만 존재 |
| **데이터 저장 위치** | 로컬 `.mv2` 파일 | 클라우드 | 로컬 Docker |
| **완전 오프라인** | ✅ 가능 | ❌ 불가 | ✅ 가능 |
| **설치 난이도** | 중간 (스킬 직접 작성) | 쉬움 | 중간 |
| **Time-Travel 디버깅** | ✅ 내장 | ❌ 없음 | ❌ 없음 |
| **하이브리드 검색** | ✅ BM25 + 벡터 | ✅ 지원 | ✅ 그래프 기반 |
| **비용** | 무료 | 유료 플랜 존재 | 무료 |

---

### **정리 및 추천**
OpenClaw에서 메모리가 필요하다면 **지금 당장 가장 쉬운 길은 Mem0 공식 플러그인**입니다. 반면 인터넷 연결 없이 완전히 로컬에서 동작하는 프라이빗 메모리가 필요하거나, 타임라인 기반 디버깅이나 `.mv2` 파일의 이식성이 매력적이라면 위의 Skills 방식으로 Memvid를 직접 연동하는 것이 충분히 가치 있는 선택입니다. 두 프로젝트 모두 빠르게 성장 중이니, 향후 공식 연동이 생길 가능성도 있습니다.



## Claude Code에서 Memvid 사용하기
결론부터 말씀드리면 **사용할 수 있습니다.** 그것도 두 가지 독립적인 방법으로 가능합니다. 하나는 **MCP 서버** 방식이고, 다른 하나는 **Agent Skills(SKILL.md)** 방식입니다. 각각 특성이 다르므로 용도에 맞게 선택하면 됩니다.

---

### **방법 1: MCP 서버 방식 (권장)**
MCP(Model Context Protocol)는 Claude Code가 외부 도구와 통신하는 표준 프로토콜입니다. `angrysky56/memvid_mcp_server` 라는 오픈소스 MCP 서버가 이미 GitHub에 존재하며, 이를 통해 Claude Code가 Memvid의 모든 기능을 도구(tool)로 직접 호출할 수 있습니다.

**설치 및 설정**

```bash
# 1. 저장소 클론
git clone https://github.com/angrysky56/memvid_mcp_server
cd memvid_mcp_server

# 2. 가상환경 생성 및 의존성 설치
uv venv --python 3.12 --seed
source .venv/bin/activate
uv add -e .
```

그 다음 Claude Code의 MCP 설정 파일에 서버를 등록합니다.

```bash
# Claude Code CLI로 MCP 서버 추가
claude mcp add memvid-memory \
  --command "uv" \
  --args "--directory /path/to/memvid_mcp_server run python memvid_mcp_server/main.py"
```

또는 설정 파일(`~/.claude/claude.json` 혹은 프로젝트 루트의 `.claude/settings.json`)에 직접 작성할 수도 있습니다.

```json
{
  "mcpServers": {
    "memvid-memory": {
      "command": "uv",
      "args": [
        "--directory",
        "/path/to/memvid_mcp_server",
        "run",
        "python",
        "memvid_mcp_server/main.py"
      ],
      "env": {
        "PYTHONWARNINGS": "ignore"
      }
    }
  }
}
```

**MCP로 제공되는 도구 목록**

MCP 서버가 연결되면 Claude Code는 아래 도구들을 자동으로 인식하고 대화 중 필요할 때 호출합니다.

| 도구명 | 기능 |
|---|---|
| `add_text` | 텍스트 문서를 메모리에 추가 |
| `add_chunks` | 텍스트 청크 목록을 한 번에 추가 |
| `add_pdf` | PDF 파일을 파싱해서 메모리에 저장 |
| `build_video` | 추가한 내용으로 `.mv2` 메모리 파일 빌드 |
| `search_memory` | 자연어로 시맨틱 검색 |
| `chat_with_memvid` | 저장된 지식 베이스와 대화 |
| `get_server_status` | 서버 상태 및 버전 확인 |

**실제 사용 예시**

Claude Code 세션 안에서 자연어로 대화하면 Claude가 알아서 적절한 도구를 호출합니다.

```
사용자: 이 프로젝트의 아키텍처 결정 사항들을 메모리에 저장해줘.
         우리는 FastAPI + PostgreSQL 구조를 쓰고, 인증은 JWT 방식이야.

Claude: (add_text 도구 호출)
        → "FastAPI + PostgreSQL 구조, JWT 인증 방식 사용" 저장 완료
        → build_video 호출하여 .mv2 파일 업데이트
```

```
사용자: 저번에 인증 방식에 대해 뭐라고 했었지?

Claude: (search_memory 도구 호출, query="인증 방식")
        → "JWT 인증 방식 사용" 검색 결과 반환
```

---

### **방법 2: Agent Skills(SKILL.md) 방식**
Claude Code는 OpenClaw와 동일한 **AgentSkills** 형식의 `SKILL.md` 파일을 지원합니다. 이 방식은 MCP 서버 없이 CLI 도구만으로 가볍게 연동할 수 있습니다. `memvid-cli`가 설치되어 있으면 바로 사용할 수 있습니다.

**디렉터리 구조**

```
~/.claude/skills/
└── memvid-memory/
    └── SKILL.md
```

**`SKILL.md` 작성 예시**

```markdown
---
name: memvid-memory
description: >
  Persistent local memory using Memvid (.mv2 file).
  Use this to save important facts, decisions, and context
  that should be remembered across coding sessions.
metadata:
  {
    "openclaw": {
      "emoji": "🧠",
      "requires": { "bins": ["memvid"] },
      "install": [
        {
          "id": "node",
          "kind": "node",
          "package": "memvid-cli",
          "bins": ["memvid"],
          "label": "Install Memvid CLI (npm)"
        }
      ]
    }
  }
---

## Memvid Persistent Memory

You have access to a local persistent memory file at `~/.claude/memory.mv2`.

### When to SAVE memory
- User explicitly says "remember this" or "save this"
- Key architectural decisions are made
- Important project constraints or rules are established
- After completing a significant milestone

### How to SAVE
```bash
echo "FACT_TO_SAVE" | memvid put ~/.claude/memory.mv2
```

### When to SEARCH memory
- At the start of a new session, proactively search for project context
- Before making architectural decisions
- When user asks about past decisions or context

### How to SEARCH
```bash
# Hybrid search (default, best quality)
memvid find ~/.claude/memory.mv2 --query "QUERY_HERE"

# Keyword search only (faster)
memvid find ~/.claude/memory.mv2 --query "QUERY" --mode lex
```

### Entity lookup (O(1), instant)
```bash
memvid state ~/.claude/memory.mv2 "ENTITY_NAME"
```

Always confirm what you saved and provide the search results in a readable format.
```
```  

---

### **두 방법의 비교**

| 항목 | **MCP 서버 방식** | **SKILL.md 방식** |
|---|---|---|
| **설치 복잡도** | 중간 (Python 환경 필요) | 쉬움 (CLI 하나면 충분) |
| **기능 완성도** | ✅ 전체 Memvid v2 API 사용 가능 | ⚠️ CLI 기능만 사용 |
| **자동 호출** | ✅ Claude가 판단해서 자동 호출 | ✅ Claude가 bash로 호출 |
| **프로젝트별 분리** | 설정 파일로 제어 | 워크스페이스별 SKILL.md |
| **v2 `.mv2` 지원** | ⚠️ 현재 서버는 v1 기반일 수 있음 | ✅ 최신 CLI 사용 |
| **오프라인 동작** | ✅ 완전 로컬 | ✅ 완전 로컬 |

---

### **실전 활용 시나리오 — 코딩 세션 장기 메모리**
가장 강력한 활용법은 **프로젝트별 지식 베이스 구축**입니다. 예를 들어 장기 프로젝트를 진행할 때, 세션이 끊겨도 아래와 같은 정보들이 `.mv2` 파일 하나에 누적됩니다.

- 팀의 코딩 컨벤션 ("우리는 single quote를 사용한다")
- 아키텍처 결정 이유 ("Redis를 캐시로 선택한 이유는 Pub/Sub 때문")
- 과거 버그의 원인과 해결법
- PR 리뷰에서 반복되는 피드백 패턴

이 `.mv2` 파일을 프로젝트 Git 저장소에 `git lfs`로 커밋하면, 팀원 모두가 동일한 AI 메모리를 공유하는 **팀 공유 AI 기억**을 만들 수 있습니다.

---

### **정리**
Claude Code에서 Memvid를 사용하는 것은 충분히 가능합니다. **MCP 서버 방식**은 가장 통합도가 높고 Claude가 자동으로 메모리를 관리하지만 Python 환경 설정이 필요합니다. **SKILL.md 방식**은 `memvid-cli` 하나만 설치하면 되므로 빠르게 시작할 수 있습니다. 공식 연동이 아닌 커뮤니티 기반 솔루션이므로, 안정성보다 기능성과 이식성을 우선시하는 프로젝트에 특히 적합합니다.
  
  
  
---
  

## 🛠️ AI Agent에서 Memvid를 Skill로 활용하는 실전 시나리오
Memvid를 AI Agent의 Skill로 붙였을 때, 핵심 가치는 한 문장으로 요약됩니다. **"에이전트가 경험을 축적하고, 그것을 다음 세션에 그대로 이어받는다."** 아래 시나리오들은 이 철학을 다양한 실제 상황에 적용한 사례들입니다.

---

### **시나리오 1 — 장기 소프트웨어 프로젝트의 팀 공유 AI 기억**

**상황:** 6개월짜리 백엔드 서비스 개발 프로젝트. 5명의 개발자가 각자 Claude Code를 사용하는데, 매번 새 세션이 열릴 때마다 "우리가 왜 Redis를 쓰기로 했더라?", "인증 미들웨어 패턴이 뭐였지?" 같은 질문을 Claude에게 반복해서 설명해야 하는 상황.

**Memvid Skill 적용 방식:** 프로젝트 루트에 `skills/team-memory/SKILL.md`를 두고, `team-knowledge.mv2` 파일을 Git LFS로 관리합니다. PR이 머지될 때마다 해당 결정 사항을 `.mv2`에 append하는 방식으로 팀 지식이 누적됩니다.

```markdown
---
name: team-memory
description: >
  팀 공유 프로젝트 지식 베이스. 아키텍처 결정, 코딩 컨벤션,
  과거 버그 해결 이력을 검색하고 저장한다.
---

## 세션 시작 시 의무 규칙
새 세션이 시작되면 반드시 아래를 실행한다:
```bash
memvid find ./team-knowledge.mv2 --query "현재 작업 컨텍스트" --mode hybrid
```

## 아키텍처 결정 저장
팀이 중요한 기술적 결정을 내릴 때 저장한다:
```bash
echo "[ADR-$(date +%Y%m%d)] DECISION: Redis 도입 / REASON: Pub-Sub 패턴 필요 / ALTERNATIVES: RabbitMQ 검토했으나 운영 복잡도로 탈락" \
  | memvid put ./team-knowledge.mv2
```

## 버그 이력 저장
버그를 해결했을 때 반드시 저장한다:
```bash
echo "[BUG-FIX] JWT 토큰 만료 처리 누락 / CAUSE: middleware 순서 오류 / FIX: auth 미들웨어를 rate-limiter 앞에 배치" \
  | memvid put ./team-knowledge.mv2
```
```
```  

**효과:** 새로운 팀원이 합류해도 `memvid find ./team-knowledge.mv2 --query "프로젝트 전체 컨텍스트"`만 실행하면 6개월치 팀 결정 이력이 즉시 복원됩니다. Claude는 이를 기반으로 팀의 기술 스택과 철학에 맞는 코드를 처음부터 생성합니다.

---

### **시나리오 2 — 코드 리뷰 학습 에이전트**

**상황:** 팀 리드가 같은 실수를 반복하는 코드를 계속 리뷰하고 있습니다. "타입 힌트 누락", "에러 핸들링 없음", "SQL 쿼리 N+1" 같은 패턴이 PR마다 반복적으로 지적되는 상황. 이 피드백들을 에이전트가 학습해서 사전에 차단할 수 있도록 하고 싶습니다.

**Memvid Skill 적용 방식:** CI 파이프라인에서 PR이 열릴 때마다 에이전트가 자동으로 `code-review-patterns.mv2`를 검색하여 팀 특화 코드 리뷰를 수행합니다.

```markdown
---
name: code-review-learner
description: >
  팀의 PR 리뷰 피드백을 학습하고, 새 PR에서 같은 패턴을 사전 탐지한다.
---

## 리뷰 피드백 학습 (PR 머지 후 자동 실행)
```bash
# GitHub Actions에서 호출되는 패턴
echo "[REVIEW-PATTERN] 파일: auth/middleware.py / 이슈: JWT 검증 없이 user_id 직접 사용 / 심각도: HIGH / 수정방향: get_current_user() 의존성 주입 사용" \
  | memvid put ./code-review-patterns.mv2
```

## 새 PR 사전 검토
```bash
# 변경된 파일 목록으로 관련 패턴 검색
git diff --name-only HEAD~1 | while read file; do
  memvid find ./code-review-patterns.mv2 \
    --query "$(head -50 $file)" --mode hybrid
done
```

## 엔티티 기반 파일별 이력 조회 (O(1))
```bash
memvid state ./code-review-patterns.mv2 "auth/middleware.py"
# → { past_issues: ["JWT 검증 누락", "rate limit 미적용"], severity: "HIGH" }
```
```

**실제 작동 흐름:**

```
1. PR #142 오픈: auth/login.py 수정
2. Agent가 자동 실행:
   memvid find code-review-patterns.mv2 --query "auth login JWT"
3. 검색 결과: "JWT 토큰 만료 처리 누락 (3회 반복 지적)"
4. Agent가 PR에 자동 코멘트:
   "⚠️ 과거 3번의 PR에서 이 파일의 JWT 만료 처리가 누락되었습니다.
    토큰 갱신 로직(refresh_token)을 반드시 포함해주세요."
5. 피드백이 실제로 반영됐을 때 → 팀 리드가 "좋은 수정"으로 확인하면
   해당 패턴을 해결된 이슈로 업데이트
```
```  
  
---

### **시나리오 3 — 개인 개발자의 다중 프로젝트 컨텍스트 전환**

**상황:** 프리랜서 개발자가 동시에 5개 프로젝트를 진행 중. 각 프로젝트마다 기술 스택, 클라이언트 요구사항, 특이 제약사항이 다름. 프로젝트 A (Django + React) 작업하다가 프로젝트 B (FastAPI + Vue)로 전환할 때마다 Claude에게 컨텍스트를 다시 설명하는 데 15분씩 소요됩니다.

**Memvid Skill 적용 방식:** 프로젝트마다 독립적인 `.mv2` 파일을 유지하고, 프로젝트 전환 시 단 한 번의 명령어로 컨텍스트를 로드합니다.

```
~/.claude/projects/
├── project-a/client-alpha.mv2
├── project-b/client-beta.mv2
├── project-c/startup-gamma.mv2
└── shared/my-coding-style.mv2   ← 공통 개인 스타일
```

```markdown
---
name: project-switcher
description: >
  다중 프로젝트 컨텍스트 전환. 현재 디렉터리 기반으로 적합한
  프로젝트 메모리를 자동 로드한다.
---

## 세션 시작 시 자동 컨텍스트 로드
```bash
# 현재 프로젝트 디렉터리에서 .mv2 파일 자동 탐색
PROJECT_MEM=$(find . -name "*.mv2" -maxdepth 2 | head -1)

if [ -n "$PROJECT_MEM" ]; then
  echo "=== 프로젝트 컨텍스트 로드 ==="
  memvid find "$PROJECT_MEM" --query "프로젝트 개요 기술스택 제약사항"
  
  echo "=== 최근 작업 이력 ==="
  memvid find "$PROJECT_MEM" --query "마지막 세션 작업내용"
fi

echo "=== 개인 코딩 스타일 ==="
memvid find ~/.claude/projects/shared/my-coding-style.mv2 \
  --query "코딩 컨벤션 선호도"
```

## 세션 종료 시 작업 요약 저장
```bash
# 세션 종료 전 오늘 작업한 내용 저장
echo "[$(date '+%Y-%m-%d')] 작업완료: $TASK_SUMMARY / 미완료: $PENDING / 다음세션: $NEXT_STEPS" \
  | memvid put "$PROJECT_MEM"
```
```
```  

**효과:** 프로젝트 B 폴더로 이동해 `claude`를 실행하는 순간, 자동으로 해당 클라이언트의 기술 스택, 지난 세션 미완료 작업, 특이 제약사항이 로드됩니다. 컨텍스트 전환 시간이 15분에서 30초로 단축됩니다.

---

### **시나리오 4 — 문서 기반 RAG 질의응답 에이전트**

**상황:** 300페이지짜리 사내 API 문서, 레거시 코드베이스 설명서, 외부 라이브러리 문서들이 흩어져 있습니다. 개발자들이 "이 엔드포인트 파라미터가 뭐였지?", "이 레거시 함수 어디서 호출되지?" 같은 질문을 반복하고 있는 상황입니다.

**Memvid Skill 적용 방식:** 모든 문서를 `.mv2` 파일로 인덱싱하고, Skill이 질문에 따라 자동으로 관련 문서를 찾아 답변합니다.

```markdown
---
name: docs-rag
description: >
  사내 문서, API 스펙, 레거시 코드 설명서를 검색한다.
  코드 작성 전 반드시 관련 문서를 먼저 확인한다.
---

## 초기 문서 인덱싱 (최초 1회 실행)
```bash
# PDF 문서 일괄 인덱싱
for pdf in ./docs/*.pdf; do
  memvid put ./company-docs.mv2 < "$pdf"
done

# 마크다운 문서 인덱싱
find ./docs -name "*.md" | while read f; do
  memvid put ./company-docs.mv2 < "$f"
done

echo "인덱싱 완료: $(memvid state ./company-docs.mv2 | grep frame_count)개 프레임"
```

## API 문서 검색
```bash
# 특정 엔드포인트 정보 검색
memvid find ./company-docs.mv2 \
  --query "/api/v2/users endpoint parameters response schema" \
  --mode hybrid

# 엔티티 직접 조회 (O(1))
memvid state ./company-docs.mv2 "POST /api/v2/users"
# → { method: POST, params: [...], auth: required, rate_limit: "100/min" }
```

## 레거시 코드 패턴 검색
```bash
memvid find ./company-docs.mv2 \
  --query "UserAuthService 클래스 사용법 의존성" \
  --mode sem   # 의미론적 검색으로 유사 패턴도 탐지
```

**실제 워크플로우:**

```
개발자: "결제 처리 API 엔드포인트 구현해줘"

Agent 내부 동작:
1. docs-rag skill 자동 발동
2. memvid find company-docs.mv2 --query "payment API endpoint spec"
3. 검색 결과: PaymentService 스펙, 인증 방식, 에러코드 목록 반환
4. 해당 스펙을 컨텍스트로 삼아 코드 생성

→ 문서와 정확히 일치하는 엔드포인트 구현 완성
→ "문서 확인해봐야 하는데..." 시간 절약
```
  
---

### **시나리오 5 — 인시던트 대응 에이전트 (Runbook 메모리)**

**상황:** 새벽 3시에 서비스 장애 발생. 온콜 엔지니어가 반쯤 잠든 상태로 대응해야 합니다. 과거 유사 인시던트의 해결 방법을 빠르게 찾고, 대응 과정을 자동으로 기록해야 하는 상황입니다.

**Memvid Skill 적용 방식:** 모든 인시던트 대응 이력과 Runbook을 `.mv2`에 저장하고, Time-Travel 기능으로 특정 시점의 시스템 상태도 재현합니다.

```markdown
---
name: incident-responder
description: >
  인시던트 대응 Runbook 검색 및 이력 기록.
  장애 발생 시 즉시 발동하여 유사 사례와 해결책을 제시한다.
---

## 인시던트 발생 시 즉각 검색
```bash
# 에러 메시지로 유사 사례 즉시 검색
INCIDENT_MSG="$1"
echo "=== 유사 인시던트 검색 중 ==="
memvid find ./incidents.mv2 \
  --query "$INCIDENT_MSG" \
  --mode hybrid

echo "=== 관련 Runbook 검색 ==="
memvid find ./runbooks.mv2 \
  --query "$INCIDENT_MSG 해결 절차" \
  --mode sem
```

## 대응 과정 실시간 기록
```bash
# 타임스탬프와 함께 대응 액션 기록
log_action() {
  echo "[$(date -u '+%Y-%m-%dT%H:%M:%SZ')] INCIDENT: $INCIDENT_ID / ACTION: $1 / RESULT: $2" \
    | memvid put ./incidents.mv2
}

log_action "DB 커넥션 풀 확인" "max_conn=100, 현재=98 (포화 상태)"
log_action "커넥션 풀 확장" "max_conn=200으로 증가 후 정상화"
log_action "RCA" "배치 작업이 커넥션 반환 안 함 - connection leak"
```

## 사후 분석 (Post-Mortem) 검색
```bash
# 특정 날짜의 인시던트 상태 재현 (Time-Travel)
memvid find ./incidents.mv2 \
  --query "DB connection pool 2026-01" \
  --mode lex  # 정확한 키워드 매칭
```
```

**Time-Travel 디버깅 활용:**

```bash
# "2주 전 새벽 3시에 에이전트가 어떤 결정을 했는지" 재현
memvid session replay ./incidents.mv2 \
  --session inc-20260210-0347 \
  --top-k 20

# → 해당 인시던트 당시의 정확한 메모리 상태와 대응 과정 재현
# → 같은 상황에서 더 나은 대응책을 사전에 Runbook에 추가 가능
```

---

### **시나리오 6 — 멀티 에이전트 지식 공유 파이프라인**

**상황:** Research Agent, Coding Agent, Review Agent가 협력하는 파이프라인. Research Agent가 찾은 정보를 Coding Agent가 활용하고, Review Agent가 검증해야 하는데, 에이전트 간에 정보가 제대로 전달되지 않는 문제가 있습니다.

**Memvid Skill 적용 방식:** 세 에이전트가 하나의 `shared-context.mv2` 파일을 공유 메모리로 사용합니다. 각 에이전트는 자신의 작업 결과를 쓰고, 다른 에이전트의 결과를 읽는 방식으로 협력합니다.

```
shared-context.mv2
├── [Research Agent 출력] 기술 스택 리서치 결과
├── [Coding Agent 입력] ← Research 결과 참조
├── [Coding Agent 출력] 구현 완료 코드 요약
├── [Review Agent 입력] ← Coding 결과 참조
└── [Review Agent 출력] 리뷰 피드백
```

```markdown
---
name: multi-agent-shared-memory
description: >
  멀티 에이전트 파이프라인의 공유 메모리.
  각 에이전트는 자신의 역할에 맞게 읽고 쓴다.
---

## Research Agent 역할
```bash
# 리서치 결과 저장
echo "[RESEARCH][$(date)] TOPIC: $TOPIC / FINDINGS: $FINDINGS / SOURCES: $SOURCES / CONFIDENCE: HIGH" \
  | memvid put ./shared-context.mv2

# 다음 에이전트를 위한 요약 태그
echo "[HANDOFF-TO-CODER] 사용 추천 라이브러리: $LIB_LIST / 주의사항: $WARNINGS" \
  | memvid put ./shared-context.mv2
```

## Coding Agent 역할
```bash
# Research 결과 먼저 확인
memvid find ./shared-context.mv2 \
  --query "HANDOFF-TO-CODER $CURRENT_TASK" --mode lex

# 구현 결과 저장
echo "[CODE-COMPLETE][$(date)] TASK: $TASK / FILES: $FILES_CHANGED / APPROACH: $APPROACH" \
  | memvid put ./shared-context.mv2
```

## Review Agent 역할
```bash
# 코딩 결과와 리서치 원문 동시 조회
memvid find ./shared-context.mv2 \
  --query "CODE-COMPLETE $TASK 보안 취약점 코드 품질" --mode hybrid

# 리뷰 결과 저장
echo "[REVIEW][$(date)] TASK: $TASK / STATUS: APPROVED / ISSUES: $ISSUES" \
  | memvid put ./shared-context.mv2
```
```

**멀티 에이전트 시퀀스 다이어그램:**

```
Research Agent                shared-context.mv2           Coding Agent
     │                               │                          │
     │── put("Python asyncio 권장") ─►│                          │
     │── put("HANDOFF-TO-CODER...") ─►│                          │
     │                               │◄── find("HANDOFF-TO-CODER async") ─│
     │                               │──────── 결과 반환 ────────────────►│
     │                               │                          │── 코드 구현
     │                               │◄── put("CODE-COMPLETE") ─────────│
     │                               │
                                Review Agent
                                     │
                               find("CODE-COMPLETE") ──────────►│
                               ◄──────── 검토 결과 ──────────────│
```
```  
  
---

### **시나리오 7 — 개인화 학습 튜터 에이전트**

**상황:** 개발자가 Rust를 독학 중. 매번 새 세션마다 "나는 Python 백엔드 개발자야, Rust는 초보야, 소유권 개념이 아직 헷갈려"라고 설명해야 하는 번거로움이 있습니다. 또한 이미 배운 개념을 다시 기초부터 설명하는 AI의 반복적인 설명이 비효율적인 상황입니다.

**Memvid Skill 적용 방식:** 학습자의 이해도, 진도, 오개념 패턴을 누적해서 완전히 개인화된 튜터 경험을 만듭니다.

```markdown
---
name: learning-tutor
description: >
  학습자의 현재 수준과 진도를 기억하는 개인화 튜터.
  이미 이해한 개념은 건너뛰고, 약점에 집중한다.
---

## 세션 시작 시 학습자 프로파일 로드
```bash
echo "=== 현재 학습 수준 ==="
memvid state ./learning-profile.mv2 "rust-ownership"
# → { status: "confused", attempts: 3, last_error: "dangling reference" }

memvid state ./learning-profile.mv2 "rust-lifetime"
# → { status: "not_started" }

echo "=== 최근 5개 학습 세션 요약 ==="
memvid find ./learning-profile.mv2 \
  --query "최근 학습 진도 성취" --mode lex
```

## 이해도 업데이트
```bash
# 개념 이해 성공 기록
update_concept() {
  CONCEPT=$1; STATUS=$2; NOTE=$3
  echo "[CONCEPT-UPDATE] concept=$CONCEPT / status=$STATUS / note=$NOTE / date=$(date)" \
    | memvid put ./learning-profile.mv2
  
  # O(1) 조회를 위한 엔티티 등록
  memvid enrich ./learning-profile.mv2 --engine rules
}

update_concept "rust-borrowing" "understood" "immutable 참조는 이해, mutable 참조 제약 아직 헷갈림"
update_concept "rust-ownership" "mastered" "move semantics 완전 이해 완료"
```

## 약점 기반 다음 학습 추천
```bash
memvid find ./learning-profile.mv2 \
  --query "confused struggling not_understood" --mode lex
# → 아직 이해 안 된 개념 목록 반환
# → Claude가 이것들을 우선 설명
```
```

**개인화 학습 진도 예시:**

```
세션 1: "Rust 소유권이 뭔가요?"
  → profile.mv2: {rust-ownership: "introduced"}

세션 5: "여전히 dangling reference가 헷갈려요"
  → profile.mv2: {rust-ownership: "struggling", attempts: 5}
  → Claude: "소유권을 3번이나 다뤘는데 아직 헷갈리시는군요.
             다른 방식으로 설명해볼게요. Python의 참조 카운팅과 비교하면..."

세션 12: 드디어 이해!
  → profile.mv2: {rust-ownership: "mastered"}
  → 이후 세션: 소유권 기초 설명 건너뛰고 바로 Lifetime으로 진행
```
```  

---

### **Skill 설계 시 공통 원칙 정리**
각 시나리오에서 공통적으로 적용되는 Memvid Skill 설계 패턴이 있습니다.

**저장 시점의 원칙**으로는 세션 종료 직전 작업 요약을 반드시 저장하고, 중요한 결정이 내려졌을 때 즉시 저장하며, 에러/버그를 해결했을 때 원인과 해결책을 함께 저장하는 것이 좋습니다.

**검색 모드 선택 기준**으로는 정확한 키워드가 있을 때(에러 메시지, 함수명, 날짜)는 `--mode lex`를 사용하고, 의미 기반 검색이 필요할 때(관련 개념, 유사 패턴)는 `--mode sem`을 사용하며, 일반적인 경우에는 기본값인 하이브리드 검색을 사용하는 것이 좋습니다.

**엔티티 설계 원칙**으로는 자주 조회하는 정보는 `memvid enrich`로 엔티티화하여 O(1) 조회를 활용하고, 파일명, 컴포넌트명, 사람 이름, API 엔드포인트 같이 명확한 식별자가 있는 정보들이 엔티티화의 좋은 후보입니다.

**파일 분리 전략**으로는 프로젝트별, 도메인별로 `.mv2` 파일을 분리하여 검색 정확도를 높이는 것이 중요합니다. 팀 공유 지식, 개인 스타일, 인시던트 이력은 서로 다른 파일로 관리하는 것이 적절합니다.

| 시나리오 | 검색 모드 | 파일 구조 | 핵심 이점 |
|---|---|---|---|
| 팀 공유 기억 | hybrid | `team-knowledge.mv2` (Git LFS) | 팀 온보딩 시간 90% 절감 |
| 코드 리뷰 학습 | lex + sem | `code-review-patterns.mv2` | 반복 실수 사전 차단 |
| 다중 프로젝트 | hybrid | 프로젝트별 독립 `.mv2` | 컨텍스트 전환 30초 |
| 문서 RAG | sem | `company-docs.mv2` | 문서 검색 즉시 답변 |
| 인시던트 대응 | lex | `incidents.mv2` + Time-Travel | 새벽 장애 대응 속도 향상 |
| 멀티 에이전트 | lex | `shared-context.mv2` (공유) | 에이전트 간 정보 손실 제거 |
| 개인화 학습 | lex | `learning-profile.mv2` | 반복 설명 제거, 약점 집중 |   