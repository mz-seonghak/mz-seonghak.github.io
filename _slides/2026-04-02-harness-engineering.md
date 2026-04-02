---
title: "Harness Engineering — AI 에이전트를 프로덕션에서 진짜로 동작하게 만드는 기술"
date: 2026-04-02
tags: [harness-engineering, ai-agent, context-engineering, llm, production, codex, claude-code]
description: "2026년 핵심 키워드 하네스 엔지니어링 — 정의, 6대 구성요소, 필수 도구, 실전 적용법, OpenAI·Anthropic 사례, 최신 트렌드를 슬라이드로 정리합니다."
theme: white
---

# Harness Engineering

AI 에이전트를 프로덕션에서 진짜로 동작하게 만드는 기술

---

## 하네스 엔지니어링이란?

> AI 에이전트가 **무엇을 보고, 무엇을 할 수 있고, 언제 멈추고, 실패하면 어떻게 되는지**를 설계하는 엔지니어링 규율

LLM은 **강력하지만 방향 감각이 없는 말** 🐴

하네스는 그 힘을 통제하는 **고삐, 안장, 재갈** 역할

---

## 프롬프트 vs 컨텍스트 vs 하네스

| 개념 | 초점 |
|------|------|
| **프롬프트 엔지니어링** | 단일 모델 호출의 품질 개선 |
| **컨텍스트 엔지니어링** | 컨텍스트 윈도우에 *무엇을* 넣을지 |
| **하네스 엔지니어링** | 윈도우 *바깥*의 모든 것 — 도구, 상태, 검증, 생명주기 |

프롬프트 = "한 번의 대화를 잘하는 법"

하네스 = "100번의 세션을 걸쳐 일관되게 잘하는 법"

---

## 왜 하네스가 필요한가?

### LLM의 근본적 한계

- **무상태(stateless)** — 매 세션이 백지 상태
- **컨텍스트 붕괴** — 윈도우가 차면 원래 지시를 놓침
- **환각적 도구 호출** — 없는 API를 참조
- **조기 완료 선언** — 검증 없이 "done" 선언

---

## 모델은 상품화되었다

Claude, GPT, Gemini, 오픈소스 모델 성능은 수렴 중

**동일 모델에서 하네스 유무에 따라 작업 완료율 40%p 차이**

> 모델은 교체 가능한 부품이고, 하네스가 곧 제품이다

---

## 6대 핵심 구성요소

1. **컨텍스트 엔지니어링** — 에이전트가 무엇을 볼 수 있는지
2. **검증 루프** — 제대로 했는지 확인
3. **상태 관리** — 어디까지 했는지 기억
4. **도구 오케스트레이션** — 무엇을 할 수 있는지 제어
5. **휴먼-인-더-루프** — 언제 사람에게 물어볼지
6. **생명주기 관리** — 시작, 끝, 세션 간 전환

---

## 1. 컨텍스트 엔지니어링

에이전트가 **무엇을 볼 수 있는지** 결정

- 코드베이스 내 지식 기반 (AGENTS.md, CLAUDE.md)
- 관찰 가능성 데이터, 동적 컨텍스트
- "Lost in the Middle" 대응 — 중요 정보를 프롬프트 시작/끝에 배치
- 3종 메모리: 작업 컨텍스트(임시) · 세션 상태(중기) · 장기 메모리(영구)

---

## 2. 검증 루프

에이전트가 **제대로 했는지** 확인

- 테스트 스위트 통과 후에만 기능 완료 표시
- 기능 목록을 JSON으로 관리 → pass/fail 기계적 추적
- 결정론적 린터 + 구조 테스트로 아키텍처 위반 감지

```json
{
  "description": "사용자가 새 대화를 생성할 수 있다",
  "passes": false
}
```

---

## 3. 상태 관리

에이전트가 **어디까지 했는지** 기억

- `claude-progress.txt` 같은 진행 로그
- Git 커밋으로 각 단계 문서화
- **JSON > Markdown** — 모델이 JSON을 덜 임의 수정함

---

## 4. 도구 오케스트레이션

에이전트가 **무엇을 할 수 있는지** 제어

- 파일 시스템, 코드 실행, API 호출, 웹 검색
- **사전 승인된 도구만** 접근 가능
- MCP(Model Context Protocol) 서버 통한 표준 인터페이스

---

## 5. 휴먼-인-더-루프

에이전트가 **언제 사람에게 물어봐야 하는지**

- 파괴적 작업(삭제, 배포)에 인간 승인 요구
- 완전 자율은 드물게 적절
- 민감한 결정에 에스컬레이션 경로 설계

---

## 6. 생명주기 관리

에이전트의 **시작과 끝, 세션 간 전환**

- 초기화 에이전트 → 코딩 에이전트 분리
- 세션 시작 표준 절차: pwd → git log → 기능 목록 → 다음 작업
- 실패 시 안전한 롤백 경로

---

## 필요한 도구들

### 코드 품질 & 제약

| 도구 | 역할 |
|------|------|
| Pre-commit hooks | 커밋 전 코드 품질 자동 검증 |
| Custom linters | 조직 고유 규칙 강제 |
| ArchUnit / 구조 테스트 | 아키텍처 제약을 테스트로 검증 |
| CI/CD 파이프라인 | 자동 빌드/테스트/배포 |

---

## 필요한 도구들

### 컨텍스트 & 메모리

| 도구 | 역할 |
|------|------|
| AGENTS.md / CLAUDE.md | 에이전트 지시사항 문서 |
| 벡터 스토어 | 장기 메모리 시맨틱 검색 |
| 진행 로그 (JSON) | 세션 간 상태 유지 |
| Git | 진행 문서화 & 롤백 |

---

## 필요한 도구들

### 도구 연결 & 검증

| 도구 | 역할 |
|------|------|
| MCP 서버 | 표준화된 도구 인터페이스 |
| Puppeteer / Playwright | 브라우저 자동화 E2E 검증 |
| 컨테이너/샌드박스 | 에이전트 실행 환경 격리 |
| 레드팀 도구 | 에이전트 취약점 사전 탐지 |

---

## SDK & 개발 킷: 코딩 에이전트 하네스

하네스가 **이미 내장된** 에이전트 환경

| 제품 | 제공사 | 핵심 하네스 기능 |
|------|--------|-----------------|
| **Claude Code** | Anthropic | 5계층 권한, 18+ 훅, CLAUDE.md, 자동 롤백 |
| **Codex CLI** | OpenAI | 샌드박스, AGENTS.md, 파일 접근 제어 |
| **Cursor** | Cursor Inc. | `.cursor/rules`, 루프 탐지, IDE 통합 |
| **Windsurf** | Codeium | Cascade 에이전트, 멀티파일 편집 |

---

## Claude Code 하네스 아키텍처

<div class="mermaid">
graph LR
    CC["Claude Code<br/>Harness"] --> PM["Permission Model<br/>5계층"]
    CC --> HK["Hooks<br/>18+ 이벤트"]
    CC --> CTX["Context<br/>CLAUDE.md + 압축 + 메모리"]
    CC --> EX["Execution<br/>서브에이전트 + 워크트리"]
    CC --> SF["Safety<br/>스냅샷 & 롤백"]
    PM --> PM1["Mode / Allowlist / MCP / Bash / User"]
    HK --> HK1["PreToolUse / PostToolUse / Session"]
</div>

---

## SDK & 개발 킷: 에이전트 개발 프레임워크

### Claude Agent SDK (Anthropic)

- **Tool-use-first** 접근 — 에이전트 = 모델 + 도구
- 훅 시스템, 권한 모델, 에이전트를 도구로 호출
- 의도적으로 단순한 설계

### OpenAI Agents SDK

- **Handoff** 메커니즘 — 에이전트 간 명시적 제어 전환
- 입력/출력 가드레일, 낙관적 실행 + 롤백
- 가장 낮은 학습 곡선

---

## SDK & 개발 킷: 에이전트 개발 프레임워크 (계속)

### Google ADK (Agent Development Kit)

- **SequentialAgent, ParallelAgent, LoopAgent** 등 명시적 워크플로우
- 에이전트를 도구로 사용하는 계층 구조
- Vertex AI 네이티브, Developer UI, 평가 도구 내장

### AWS Strands Agents SDK

- **모델 주도(model-driven)** — Model + Tools + Prompt
- 내장 도구 20+, MCP 서버 연결, `@tool` 데코레이터
- 복잡한 오케스트레이션 불필요, Bedrock 네이티브

---

## SDK & 개발 킷: 오케스트레이션 프레임워크

| 프레임워크 | 핵심 특징 | 하네스 기능 |
|-----------|----------|------------|
| **LangGraph** | 그래프 기반 상태 관리 | 체크포인트 복구, HITL, LangSmith |
| **CrewAI** | 역할 기반 멀티 에이전트 | Flows 이벤트 파이프라인 |
| **MS AutoGen** | 대화 기반 멀티 에이전트 | 그룹 채팅, 코드 실행 샌드박스 |
| **Bedrock Agents** | 완전 관리형 서비스 | 가드레일 내장, IAM, KB RAG |

---

## 도구 연결 표준: MCP

**Model Context Protocol** — 모든 SDK를 관통하는 표준

- 어떤 SDK든 **동일한 도구 인터페이스** 제공
- Claude Code, Cursor, Codex 모두 지원
- 한 번 만든 MCP 서버를 여러 에이전트에서 재사용

<div class="mermaid">
graph TB
    Agent["AI Agent<br/>(SDK 무관)"] <-->|"MCP 표준"| Server["MCP Server"]
    Server --> FS["파일시스템"]
    Server --> DB["데이터베이스"]
    Server --> API["외부 API"]
</div>

---

## SDK 선택 가이드

| 시나리오 | 추천 | 이유 |
|---------|------|------|
| 즉시 코딩 에이전트 | **Claude Code** / **Codex** | 하네스 내장 |
| 커스텀 에이전트 (Anthropic) | **Claude Agent SDK** | 단순 + 강력한 훅 |
| 커스텀 에이전트 (OpenAI) | **OpenAI Agents SDK** | 최소 학습 곡선 |
| GCP 생태계 | **Google ADK** | Vertex AI 네이티브 |
| AWS 생태계 | **Strands Agents** | Bedrock 네이티브 |
| 복잡한 멀티 에이전트 | **LangGraph** | 벤더 중립 그래프 |
| 대화 기반 에이전트 팀 | **MS AutoGen** | 그룹 채팅 패턴 |
| 관리형 서비스 | **Bedrock Agents** | 코드 최소 |

---

## 실전 아키텍처 패턴

### 패턴 A: 단일 에이전트 + 감독자 루프

하나의 모델이 도구·메모리·검증과 함께 루프

→ 고객 지원, 단순 자동화에 적합

### 패턴 B: 초기화-실행자 분할 (Anthropic 추천)

초기화 에이전트가 환경 세팅 → 코딩 에이전트가 증분 진행

→ 장기 실행 코딩 작업에 적합

### 패턴 C: 멀티 에이전트 조율

연구자·작가·검토자 등 전문가 에이전트 간 작업 위임

→ 복잡한 프로젝트, 다단계 파이프라인에 적합

---

## 실전 Step 1: 기능 목록 정의

```json
[
  {
    "category": "functional",
    "description": "사용자가 새 대화를 생성할 수 있다",
    "passes": false
  },
  {
    "category": "functional",
    "description": "메시지 전송 시 실시간 응답 표시",
    "passes": false
  }
]
```

모든 기능을 `false`로 시작 → **완료 기준을 명확히**

---

## 실전 Step 2: 초기화 스크립트

```bash
#!/bin/bash
# init.sh — 에이전트가 매 세션 시작 시 실행
npm install
npm run dev &
echo "Development server started"
```

---

## 실전 Step 3: 에이전트 지시사항

```markdown
# AGENTS.md

## 작업 규칙
- 한 번에 하나의 기능만 작업할 것
- 기능 완료 전 반드시 E2E 테스트 실행
- 테스트를 삭제하거나 수정하는 것은 금지
- 작업 완료 후 git commit으로 진행 기록
```

---

## 실전 Step 4: 검증 자동화

```yaml
# .github/workflows/agent-verify.yml
on: [push]
jobs:
  verify:
    steps:
      - run: npm run lint        # 커스텀 린터
      - run: npm run test:arch   # 구조 테스트
      - run: npm run test:e2e    # E2E 테스트
```

---

## 실전 Step 5: 엔트로피 관리

주기적으로 별도 에이전트가 코드베이스 스캔:

- 문서화 불일치 감지 및 수정
- 아키텍처 제약 위반 탐지
- 코드 부패(code rot) 식별

> **에이전트가 어려움을 겪을 때, 그것을 신호로 해석하라.**
>
> 부족한 도구, 가드레일, 문서를 식별하고 저장소에 피드백하는 것이 하네스 엔지니어링의 핵심 루프다. — OpenAI Codex 팀

---

## 사례: OpenAI Codex

- **5개월** 프로덕션 애플리케이션 개발
- **100만 줄 이상** 코드 — 전부 AI 생성
- 인간 엔지니어는 **코드 0줄**, 시스템 설계에 집중

### 3가지 핵심 전략

1. **컨텍스트 엔지니어링** — 지속적으로 개선되는 지식 기반
2. **아키텍처 제약** — 결정론적 린터 + 구조 테스트
3. **가비지 컬렉션 에이전트** — 코드베이스 엔트로피 자동 수정

> "제약이 클수록 신뢰성이 높다"

---

## 사례: Anthropic 장기 실행 에이전트

### 초기화-실행자 분할 패턴

**Session 0** (초기화): init.sh 작성 → features.json → 초기 커밋

**Session 1..N** (실행): 진행 로그 읽기 → 다음 기능 선택 → 구현 & 테스트 → 커밋

### 주요 실패 패턴

| 문제 | 해결책 |
|------|--------|
| 조기 완료 선언 | 기능 목록 + E2E 테스트 강제 |
| 세션 간 정보 손실 | Git 커밋 + 진행 로그 의무화 |
| 매번 환경 셋업 | init.sh로 표준화 |
| 기존 코드 파괴 | 이전 커밋 로그 참조 의무화 |

---

## 트렌드 1: 하네스 = 새로운 경쟁 우위

모델 성능 수렴 → **하네스 품질 = 제품 품질**

동일 모델에서 하네스 유무에 따라 **2~5배 신뢰성 차이**

---

## 트렌드 2: 새로운 직군의 탄생

**"Harness Engineer"** — 에이전트 기반 제품 회사에서 독립 역할로 등장

소프트웨어 엔지니어링 + AI 시스템 설계 + 안전성 엔지니어링의 교차점

---

## 트렌드 3: Meta-Harness

하네스 코드 자체를 **AI가 최적화**하는 개념

소스 코드, 점수, 실행 트레이스 분석 → 더 나은 하네스를 자동 제안

---

## 트렌드 4: 기술 스택 수렴

개발자 취향보다 **AI 친화적 구조**가 우선

하네스가 유지보수하기 좋은 코드 구조가 사실상의 표준

---

## 트렌드 5: 모델 드리프트 감지

하네스가 **100번째 스텝 이후 모델이 지시를 따르지 않는 시점**을 정확히 감지하는 도구로 진화

---

## 트렌드 6: 신규 vs 레거시

- **신규 프로젝트** — 처음부터 하네스 고려 설계
- **레거시 프로젝트** — 엔트로피가 높으면 레트로핏 비용 > 효과

---

## 핵심 요약

| 원칙 | 설명 |
|------|------|
| **모델은 부품, 하네스가 제품** | 모델은 교체 가능, 하네스는 경쟁 우위 |
| **제약이 신뢰를 만든다** | 해결 공간을 좁힐수록 결과 안정 |
| **실패를 신호로** | 에이전트 실패 = 하네스 개선 기회 |
| **검증은 자동으로** | E2E 테스트와 구조 테스트 |
| **점진적 진행** | 한 번에 하나, 매번 커밋 |

---

## 참고 자료

- [Harness Engineering — Martin Fowler](https://martinfowler.com/articles/exploring-gen-ai/harness-engineering.html)
- [Harness Engineering — OpenAI](https://openai.com/index/harness-engineering/)
- [Effective Harnesses — Anthropic](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents)
- [What Is an Agent Harness? — Firecrawl](https://www.firecrawl.dev/blog/what-is-an-agent-harness)
- [AI Harness Engineering Guide — Medium](https://medium.com/be-open/what-is-ai-harness-engineering-your-guide-to-controlling-autonomous-systems-30c9c8d2b489)
- [AI Agent Frameworks Comparison — Roberto Infante](https://medium.com/@roberto.g.infante/the-state-of-ai-agent-frameworks-comparing-langgraph-openai-agent-sdk-google-adk-and-aws-d3e52a497720)
- [Claude Agent SDK — Anthropic](https://platform.claude.com/docs/en/agent-sdk/overview)

---

# Thank You
