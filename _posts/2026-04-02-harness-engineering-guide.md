---
layout: post
title: "Harness Engineering — AI 에이전트를 프로덕션에서 진짜로 동작하게 만드는 기술"
date: 2026-04-02
tags: [harness-engineering, ai-agent, context-engineering, llm, production, codex, claude-code]
---

2025년이 AI 에이전트의 해였다면, 2026년은 **하네스(Harness)**의 해입니다. 모델은 이미 충분히 강력해졌고, 이제 경쟁력은 모델 자체가 아니라 **모델을 감싸는 시스템**에서 갈립니다.

이 글에서는 하네스 엔지니어링의 개념부터 필요한 도구, 실전 적용법, 그리고 현재 트렌드까지 슬라이드 형식으로 정리합니다.

---

## Slide 1. 하네스 엔지니어링이란?

### 한 줄 정의

> AI 에이전트가 **무엇을 보고, 무엇을 할 수 있고, 언제 멈추고, 실패하면 어떻게 되는지**를 설계하는 엔지니어링 규율

### 비유: 말과 마구(馬具)

LLM은 **강력하지만 방향 감각이 없는 말**입니다. 하네스는 그 힘을 통제 가능한 작업으로 전환하는 **고삐, 안장, 재갈** 역할을 합니다.

| 개념 | 초점 |
|------|------|
| **프롬프트 엔지니어링** | 단일 모델 호출의 품질 개선 |
| **컨텍스트 엔지니어링** | 컨텍스트 윈도우에 *무엇을* 넣을지 결정 |
| **하네스 엔지니어링** | 컨텍스트 윈도우 *바깥*의 모든 것 — 도구, 상태, 검증, 생명주기 |

하네스 엔지니어링은 프롬프트 엔지니어링의 상위 개념입니다. 프롬프트가 "한 번의 대화를 잘하는 법"이라면, 하네스는 "100번의 세션을 걸쳐 일관되게 잘하는 법"입니다.

---

## Slide 2. 왜 하네스가 필요한가?

### LLM의 근본적 한계

LLM은 기본적으로 **무상태(stateless)**입니다. 매 세션은 이전 작업에 대한 기억 없이 시작됩니다.

| 문제 | 설명 |
|------|------|
| **컨텍스트 붕괴** | 도구 결과와 이력으로 윈도우가 채워지면 원래 지시를 놓침 |
| **환각적 도구 호출** | 존재하지 않는 API를 참조하거나 잘못된 매개변수 사용 |
| **실패 시 상태 손실** | 네트워크 오류 시 진행 상황이 완전히 소실 |
| **조기 완료 선언** | 검증 없이 "완료"를 선언하는 경향 |

### 모델은 상품화되었다

Claude, GPT, Gemini, 오픈소스 모델들의 성능 차이는 좁아지고 있습니다. **동일한 모델을 사용해도 하네스 품질에 따라 작업 완료율이 40%p 차이**가 납니다.

> 모델은 교체 가능한 부품이고, 하네스가 곧 제품이다.

---

## Slide 3. 하네스의 6대 핵심 구성요소

```
┌─────────────────────────────────────────────────┐
│                  Agent Harness                   │
│                                                  │
│  ┌──────────────┐  ┌──────────────────────────┐  │
│  │ 1. Context    │  │ 2. Verification Loops    │  │
│  │  Engineering  │  │   (검증 루프)             │  │
│  └──────────────┘  └──────────────────────────┘  │
│  ┌──────────────┐  ┌──────────────────────────┐  │
│  │ 3. State      │  │ 4. Tool Orchestration    │  │
│  │  Management  │  │   (도구 오케스트레이션)     │  │
│  └──────────────┘  └──────────────────────────┘  │
│  ┌──────────────┐  ┌──────────────────────────┐  │
│  │ 5. Human-in  │  │ 6. Lifecycle Management  │  │
│  │  -the-Loop   │  │   (생명주기 관리)          │  │
│  └──────────────┘  └──────────────────────────┘  │
│                                                  │
│                ┌──────────┐                      │
│                │   LLM    │                      │
│                └──────────┘                      │
└─────────────────────────────────────────────────┘
```

### 1. 컨텍스트 엔지니어링

에이전트가 **무엇을 볼 수 있는지** 결정합니다.

- 코드베이스 내 지속적으로 개선되는 **지식 기반** (AGENTS.md, CLAUDE.md 등)
- 관찰 가능성 데이터, 브라우저 네비게이션 같은 **동적 컨텍스트**
- "Lost in the Middle" 문제 대응 — 가장 중요한 정보를 프롬프트의 시작과 끝에 배치
- 3종 메모리 운영: 작업 컨텍스트(임시), 세션 상태(중기), 장기 메모리(영구)

### 2. 검증 루프

에이전트가 **제대로 했는지** 확인합니다.

- 코딩 에이전트: 테스트 스위트 통과 후에만 기능 완료 표시
- 기능 목록을 JSON으로 관리 → 각 기능의 pass/fail 상태를 기계적으로 추적
- 결정론적 린터 + 구조 테스트로 아키텍처 제약 위반 감지

### 3. 상태 관리

에이전트가 **어디까지 했는지** 기억합니다.

- `claude-progress.txt` 같은 진행 로그
- Git 커밋으로 각 단계의 진행 상황 문서화
- JSON 형식 선호 (모델이 마크다운보다 JSON을 덜 임의로 수정함)

### 4. 도구 오케스트레이션

에이전트가 **무엇을 할 수 있는지** 제어합니다.

- 파일 시스템 접근, 코드 실행, API 호출, 웹 검색 등
- 사전 승인된 도구만 접근 가능하도록 제한
- MCP(Model Context Protocol) 서버를 통한 도구 연결

### 5. 휴먼-인-더-루프

에이전트가 **언제 사람에게 물어봐야 하는지** 결정합니다.

- 파괴적 작업(삭제, 배포 등)에 대한 인간 승인 요구
- 완전 자율은 드물게 적절 — 대부분의 시나리오에서 인간 개입 지점 설계 필수
- 민감한 결정에 대한 에스컬레이션 경로

### 6. 생명주기 관리

에이전트의 **시작과 끝, 그리고 세션 간 전환**을 관리합니다.

- 초기화 에이전트 → 코딩 에이전트 분리 패턴
- 각 세션 시작 시 표준 절차: `pwd` 확인 → git 로그 읽기 → 기능 목록 확인 → 다음 작업 선택
- 실패 시 안전한 롤백 경로

---

## Slide 4. 하네스 엔지니어링에 필요한 도구들

### 코드 품질 & 제약 도구

| 도구 | 역할 |
|------|------|
| **Pre-commit hooks** | 커밋 전 코드 품질 자동 검증 |
| **Custom linters** | 조직 고유 규칙 강제 적용 |
| **ArchUnit / 구조 테스트** | 코드 아키텍처 제약을 테스트로 검증 |
| **CI/CD 파이프라인** | 에이전트 생성 코드의 자동 빌드/테스트/배포 |

### 컨텍스트 & 메모리 도구

| 도구 | 역할 |
|------|------|
| **AGENTS.md / CLAUDE.md** | 코드베이스 내 에이전트 지시사항 문서 |
| **벡터 스토어** | 장기 메모리 저장 및 시맨틱 검색 |
| **진행 로그 (JSON)** | 세션 간 상태 유지 |
| **Git** | 진행 상황 문서화 & 롤백 지점 |

### 도구 연결 & 오케스트레이션

| 도구 | 역할 |
|------|------|
| **MCP 서버** | 표준화된 도구 인터페이스 제공 |
| **Puppeteer / Playwright** | 브라우저 자동화를 통한 E2E 검증 |
| **Firecrawl** | 웹 검색/스크래핑 — 에이전트의 웹 접근 계층 |
| **컨테이너/샌드박스** | 에이전트 실행 환경 격리 |

### 검증 & 안전 도구

| 도구 | 역할 |
|------|------|
| **테스트 프레임워크** | 기능 완료 여부 기계적 검증 |
| **레드팀 도구** | 에이전트 취약점 사전 탐지 |
| **모니터링/알림** | 에이전트 행동 이상 감지 |
| **감사 로그** | 에이전트 행동의 추적 가능성 보장 |

---

## Slide 5. 실전: 하네스 엔지니어링은 어떻게 하는가?

### 아키텍처 패턴 선택

**패턴 A: 단일 에이전트 + 감독자 루프**
- 하나의 모델이 도구·메모리·검증과 함께 루프
- 적합: 고객 지원, 단순 자동화

**패턴 B: 초기화-실행자 분할** (Anthropic 추천)
- 초기화 에이전트가 환경 세팅 후, 코딩 에이전트가 증분 진행
- 적합: 장기 실행 코딩 작업

**패턴 C: 멀티 에이전트 조율**
- 연구자·작가·검토자 등 전문가 에이전트 간 작업 위임
- 적합: 복잡한 프로젝트, 다단계 파이프라인

### 실전 체크리스트

#### Step 1: 기능 목록 정의

```json
[
  {
    "category": "functional",
    "description": "사용자가 새 대화를 생성할 수 있다",
    "passes": false
  },
  {
    "category": "functional",
    "description": "메시지 전송 시 실시간 응답이 표시된다",
    "passes": false
  }
]
```

모든 기능을 `false`로 시작하여 **완료 기준을 명확히** 합니다.

#### Step 2: 초기화 스크립트 작성

```bash
#!/bin/bash
# init.sh — 에이전트가 매 세션 시작 시 실행
npm install
npm run dev &
echo "Development server started"
```

#### Step 3: 에이전트 지시사항 문서 작성

```markdown
# AGENTS.md

## 작업 규칙
- 한 번에 하나의 기능만 작업할 것
- 기능 완료 전 반드시 E2E 테스트를 실행할 것
- 테스트를 삭제하거나 수정하는 것은 금지
- 작업 완료 후 git commit으로 진행 상황을 기록할 것

## 세션 시작 절차
1. pwd로 현재 디렉토리 확인
2. git log로 최근 진행 상황 확인
3. features.json에서 다음 미완료 기능 선택
4. init.sh로 개발 서버 시작
```

#### Step 4: 검증 자동화 구축

```yaml
# .github/workflows/agent-verify.yml
on: [push]
jobs:
  verify:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm ci
      - run: npm run lint        # 커스텀 린터
      - run: npm run test:arch   # 구조 테스트
      - run: npm run test:e2e    # E2E 테스트
```

#### Step 5: 엔트로피 관리 (가비지 컬렉션)

주기적으로 별도 에이전트가 코드베이스를 스캔합니다:

- 문서화 불일치 감지 및 수정
- 아키텍처 제약 위반 탐지
- 코드 부패(code rot) 식별

> OpenAI 팀의 핵심 통찰: **에이전트가 어려움을 겪을 때, 그것을 신호로 해석하라.** 부족한 도구, 가드레일, 문서를 식별하고 저장소에 피드백하는 것이 하네스 엔지니어링의 핵심 루프다.

---

## Slide 6. 하네스 엔지니어링을 위한 SDK & 개발 킷

하네스를 직접 처음부터 만들 필요는 없습니다. 주요 벤더와 오픈소스 커뮤니티가 에이전트 하네스 구축을 위한 SDK와 프레임워크를 제공합니다.

### 코딩 에이전트 하네스 (완성형)

이미 하네스가 내장된 에이전트 환경으로, 즉시 사용 가능합니다.

| 제품 | 제공사 | 핵심 하네스 기능 |
|------|--------|-----------------|
| **Claude Code** | Anthropic | 5계층 권한 모델, 18+ 훅 이벤트, CLAUDE.md 컨텍스트, 자동 스냅샷/롤백, 서브에이전트 스포닝, 워크트리 격리 |
| **Codex CLI** | OpenAI | 샌드박스 실행, AGENTS.md 지시사항, 파일 접근 제어, 도구 정의, 자동 검증 루프 |
| **Cursor** | Cursor Inc. | `.cursor/rules` 기반 하네스, IDE 통합, 루프 탐지, 모델별 프롬프트 적응 |
| **Windsurf** | Codeium | Cascade 에이전트, 컨텍스트 인식 코드 생성, 멀티파일 편집, 터미널 통합 |

#### Claude Code 하네스 아키텍처 상세

Claude Code는 가장 정교한 하네스 시스템 중 하나입니다:

```
Claude Code Harness
├── Permission Model (5계층)
│   ├── Mode (plan/autoEdit/fullAuto)
│   ├── Allowlist (사전 승인 패턴)
│   ├── MCP server permissions
│   ├── Bash permission rules
│   └── User prompt (최종 승인)
├── Hooks System (18+ 이벤트)
│   ├── PreToolUse / PostToolUse
│   ├── SessionStart / SessionEnd
│   ├── Notification / Stop
│   └── SubagentStop
├── Context Engineering
│   ├── CLAUDE.md (프로젝트 지시사항)
│   ├── 자동 컨텍스트 압축
│   └── 3종 메모리 (작업/세션/장기)
├── Execution
│   ├── 서브에이전트 스포닝
│   ├── 워크트리 격리 (병렬 실행)
│   └── 태스크 의존성 그래프
└── Safety
    ├── 자동 스냅샷 & 롤백
    ├── 읽기 전용 기본값
    └── 파괴적 작업 승인 요구
```

### 에이전트 개발 SDK (프레임워크)

에이전트를 직접 구축할 때 하네스 기능을 제공하는 SDK입니다.

#### Claude Agent SDK (Anthropic)

- **접근 방식**: Tool-use-first — 에이전트 = Claude 모델 + 도구
- **하네스 기능**: 훅 시스템, 권한 모델, 다른 에이전트를 도구로 호출
- **아키텍처**: 프롬프트 수신 → 필요시 도구 호출 → 구조화된 응답 반환
- **강점**: 의도적으로 단순한 설계, Anthropic 플랫폼 네이티브 통합

```python
from claude_agent_sdk import Agent, Tool

agent = Agent(
    model="claude-sonnet-4-6",
    tools=[filesystem, code_runner, browser],
    hooks={"pre_tool_use": lint_check, "post_tool_use": test_runner},
    permissions={"file_write": "ask", "bash": "restricted"}
)
```

#### OpenAI Agents SDK

- **접근 방식**: 최소주의 — 핵심 추상화는 "Handoff"
- **하네스 기능**: 입력/출력 가드레일, 낙관적 실행 + 롤백, 추적/평가
- **아키텍처**: 에이전트 간 명시적 제어 전환 (Handoff 메커니즘)
- **강점**: 가장 낮은 학습 곡선, 빠른 프로토타이핑

```python
from agents import Agent, Runner, InputGuardrail

agent = Agent(
    name="code_agent",
    instructions="...",
    tools=[file_tool, shell_tool],
    input_guardrails=[safety_check],
    output_guardrails=[quality_check]
)
result = Runner.run(agent, "Fix the login bug")
```

#### Google ADK (Agent Development Kit)

- **접근 방식**: 명시적 워크플로우 타입 (Sequential, Parallel, Loop)
- **하네스 기능**: 에이전트를 도구로 사용하는 계층 구조, Vertex AI 통합, Developer UI
- **아키텍처**: 결정론적 + 동적 흐름 혼합, OpenAPI 자동 변환
- **강점**: GCP 생태계 네이티브, 풍부한 내장 도구, 평가 도구 내장

```python
from google.adk import Agent, SequentialAgent, ParallelAgent

researcher = Agent(name="researcher", tools=[search, scrape])
writer = Agent(name="writer", tools=[file_write])
pipeline = SequentialAgent(
    agents=[researcher, writer],
    guardrails=safety_filter
)
```

#### AWS Strands Agents SDK

- **접근 방식**: 모델 주도(model-driven) — Model + Tools + Prompt
- **하네스 기능**: 내장 도구 20+, MCP 서버 연결, `@tool` 데코레이터
- **아키텍처**: 모델이 계획·도구 선택·결과 반영을 모두 주도
- **강점**: AWS Bedrock 네이티브, 복잡한 오케스트레이션 불필요

```python
from strands import Agent
from strands.tools import tool

@tool
def deploy(service: str) -> str:
    """Deploy a service to production"""
    return run_deploy(service)

agent = Agent(tools=[deploy])
result = agent("Deploy the user service")
```

### 오케스트레이션 프레임워크

에이전트 간 조율과 복잡한 워크플로우를 위한 프레임워크입니다.

| 프레임워크 | 핵심 특징 | 하네스 관련 기능 |
|-----------|----------|-----------------|
| **LangGraph** | 그래프 기반 상태 관리, 노드/엣지 명시적 정의 | 체크포인트 복구, HITL 내장, LangSmith 가드레일 |
| **CrewAI** | 역할 기반 멀티 에이전트 협업 | Flows 이벤트 파이프라인, 에이전트 간 위임 |
| **MS AutoGen** | 대화 기반 멀티 에이전트 (Microsoft) | 그룹 채팅 패턴, 코드 실행 샌드박스, 비동기 메시징 |
| **AWS Bedrock Agents** | 완전 관리형 서비스 | 사전 구축 가드레일, IAM 통합, Knowledge Base RAG |

#### Microsoft AutoGen

- **접근 방식**: 대화 기반 멀티 에이전트 — 에이전트들이 그룹 채팅처럼 대화
- **하네스 기능**: 코드 실행 샌드박스, 비동기 메시징, 그룹 채팅 관리자
- **아키텍처**: AssistantAgent, UserProxyAgent 등 역할별 에이전트가 대화로 협업
- **강점**: Microsoft 생태계 통합, 인간 참여가 자연스러운 대화 흐름에 녹아듦

```python
from autogen import AssistantAgent, UserProxyAgent, GroupChat

coder = AssistantAgent("coder", llm_config=llm_config)
reviewer = AssistantAgent("reviewer", llm_config=llm_config)
executor = UserProxyAgent("executor", code_execution_config={"work_dir": "workspace"})

group_chat = GroupChat(agents=[coder, reviewer, executor], messages=[])
```

### 도구 연결 표준: MCP (Model Context Protocol)

모든 SDK를 관통하는 핵심 표준이 **MCP**입니다.

```
┌──────────────┐     MCP     ┌──────────────┐
│  AI Agent    │ ◄─────────► │  MCP Server  │
│  (SDK 무관)   │  표준 프로토콜  │  (도구 제공)   │
└──────────────┘             └──────────────┘
                                    │
                    ┌───────────────┼───────────────┐
                    │               │               │
              ┌─────┴─────┐  ┌─────┴─────┐  ┌─────┴─────┐
              │ 파일시스템  │  │  데이터베이스 │  │  외부 API  │
              └───────────┘  └───────────┘  └───────────┘
```

- 어떤 SDK를 사용하든 동일한 도구 인터페이스 제공
- Claude Code, Cursor, Codex 모두 MCP 지원
- 한 번 만든 MCP 서버를 여러 에이전트에서 재사용

### SDK 선택 가이드

| 시나리오 | 추천 SDK | 이유 |
|---------|---------|------|
| 즉시 코딩 에이전트 사용 | **Claude Code** / **Codex CLI** | 하네스 내장, 설정만으로 시작 |
| 커스텀 에이전트 구축 (Anthropic) | **Claude Agent SDK** | 단순한 설계, 강력한 훅/권한 |
| 커스텀 에이전트 구축 (OpenAI) | **OpenAI Agents SDK** | 최소 학습 곡선, Handoff 패턴 |
| GCP 생태계 활용 | **Google ADK** | Vertex AI 네이티브, 평가 도구 |
| AWS 생태계 활용 | **Strands Agents** | Bedrock 네이티브, 모델 주도 |
| 복잡한 멀티 에이전트 파이프라인 | **LangGraph** | 그래프 기반 상태, 벤더 중립 |
| 역할 기반 에이전트 팀 | **CrewAI** | 직관적 역할 정의, Flows |
| 관리형 서비스 선호 | **AWS Bedrock Agents** | 코드 최소, 가드레일 내장 |

### 비교 요약

| 항목 | Claude Agent SDK | OpenAI Agents SDK | Google ADK | Strands | LangGraph |
|------|-----------------|-------------------|-----------|---------|-----------|
| 학습 곡선 | 낮음 | 가장 낮음 | 높음 | 낮음 | 중간 |
| 모델 유연성 | Anthropic 중심 | OpenAI 중심 | 멀티모델 | Bedrock 중심 | 벤더 중립 |
| 가드레일 | 훅 기반 | 입출력 객체 | 콜백 + 필터 | 도구 제한 | LangSmith |
| HITL | 권한 모델 내장 | 코드 구현 | 콜백 지원 | 코드 구현 | 내장 지원 |
| 배포 | 자유 | 자유 | GCP 최적 | AWS 최적 | 자유 |
| 멀티 에이전트 | 서브에이전트 | Handoff | 계층 구조 | 멀티에이전트 | 그래프 노드 |

---

## Slide 7. 실제 사례: OpenAI Codex

OpenAI의 Codex 팀은 하네스 엔지니어링의 가장 대표적인 사례입니다.

### 프로젝트 개요
- **5개월** 동안 프로덕션 애플리케이션 개발
- **100만 줄 이상**의 코드, 전부 AI가 생성
- 인간 엔지니어는 **코드를 한 줄도 직접 작성하지 않음**
- 엔지니어의 역할: **AI가 코드를 안정적으로 작성할 수 있는 시스템을 설계**

### 3가지 핵심 하네스 전략

**1. 컨텍스트 엔지니어링**
- 코드베이스 내 지속적으로 개선되는 지식 기반
- 관찰 가능성 데이터와 동적 컨텍스트 제공

**2. 아키텍처 제약**
- 결정론적 커스텀 린터와 구조 테스트
- LLM 기반 방식과 결정론적 방식의 혼합
- 에이전트의 "해결 공간"을 좁혀서 신뢰성 확보

**3. 가비지 컬렉션 에이전트**
- 주기적으로 코드베이스 엔트로피를 감지·수정
- 문서 불일치, 아키텍처 위반, 코드 부패 자동 탐지

### 핵심 교훈

> "제약이 클수록 신뢰성이 높다" — 무제한 유연성보다 **제약된 해결 공간**이 더 나은 결과를 만든다

---

## Slide 8. 실제 사례: Anthropic의 장기 실행 에이전트 하네스

Anthropic은 장기 실행 코딩 에이전트를 위한 **초기화-실행자 분할 패턴**을 권장합니다.

### 2단계 아키텍처

```
Session 0 (초기화)          Session 1..N (실행)
┌────────────────┐         ┌────────────────┐
│ 초기화 에이전트  │         │ 코딩 에이전트    │
│                │         │                │
│ • init.sh 작성  │         │ • 진행 로그 읽기  │
│ • features.json│   →     │ • 다음 기능 선택  │
│ • 초기 커밋     │         │ • 구현 & 테스트   │
│ • 환경 검증     │         │ • 커밋 & 업데이트  │
└────────────────┘         └────────────────┘
```

### 주요 실패 패턴과 해결책

| 문제 | 원인 | 해결책 |
|------|------|--------|
| 조기 완료 선언 | 검증 없이 "done" | 기능 목록 + E2E 테스트 강제 |
| 문서화 부족 | 세션 간 정보 손실 | Git 커밋 + 진행 로그 의무화 |
| 앱 실행에 시간 낭비 | 매번 환경 셋업 | init.sh로 표준화 |
| 기존 코드 파괴 | 컨텍스트 부족 | 이전 커밋 로그 참조 의무화 |

### 베스트 프랙티스

- **JSON > Markdown**: 모델이 JSON 파일은 임의로 수정하는 경향이 적음
- **강력한 금지 지시**: "테스트를 삭제하거나 수정하는 것은 허용되지 않음"
- **E2E 검증 우선**: 단위 테스트보다 실제 사용자 관점의 E2E 테스트

---

## Slide 9. 현재 트렌드와 미래 방향

### 트렌드 1: 하네스 = 새로운 경쟁 우위

모델 성능이 수렴하면서, **하네스 품질이 곧 제품 품질**이 되었습니다. 동일 모델에서 하네스 유무에 따라 **2~5배의 신뢰성 차이**가 발생합니다.

### 트렌드 2: 하네스 엔지니어, 새로운 직군의 탄생

에이전트 기반 제품을 만드는 회사에서 "Harness Engineer"가 독립적인 역할로 등장하고 있습니다. 기존 소프트웨어 엔지니어링 + AI 시스템 설계 + 안전성 엔지니어링의 교차점입니다.

### 트렌드 3: Meta-Harness — 하네스를 최적화하는 AI

최신 연구에서는 **하네스 코드 자체를 AI가 최적화**하는 메타 하네스 개념이 등장했습니다. 소스 코드, 점수, 실행 트레이스를 분석하여 더 나은 하네스를 자동으로 제안합니다.

### 트렌드 4: 기술 스택의 수렴

개발자의 프레임워크/언어 취향보다 **AI 친화적 구조**가 우선시되는 경향입니다. 하네스가 유지보수하기 좋은 코드 구조가 사실상의 표준으로 자리잡고 있습니다.

### 트렌드 5: 모델 드리프트 감지

하네스가 **모델이 100번째 스텝 이후 지시를 따르지 않거나 추론 오류를 범하는 시점**을 정확히 감지하는 도구로 진화하고 있습니다.

### 트렌드 6: 신규 vs 기존 코드베이스 분화

- **신규 프로젝트**: 처음부터 하네스를 고려하여 설계
- **기존 프로젝트**: 하네스 레트로핏이 항상 가치 있지는 않음 — 엔트로피가 높은 레거시 코드는 비용 대비 효과가 낮을 수 있음

---

## Slide 10. 핵심 요약

```
┌────────────────────────────────────────────────────────┐
│                                                        │
│   "더 나은 모델"이 아니라 "더 나은 제어 환경"이         │
│    장기적 코드 품질과 에이전트 신뢰성을 결정한다        │
│                                                        │
└────────────────────────────────────────────────────────┘
```

| 원칙 | 설명 |
|------|------|
| **모델은 부품, 하네스가 제품** | 모델은 교체 가능하지만 하네스는 경쟁 우위 |
| **제약이 신뢰를 만든다** | 해결 공간을 좁힐수록 결과가 안정적 |
| **실패를 신호로** | 에이전트 실패 → 하네스 개선 기회 |
| **검증은 자동으로** | 수동 검토 대신 E2E 테스트와 구조 테스트 |
| **점진적 진행** | 한 번에 하나의 기능, 매번 커밋 |

---

## 참고 자료

- [Harness Engineering — Martin Fowler](https://martinfowler.com/articles/exploring-gen-ai/harness-engineering.html)
- [Harness Engineering: Leveraging Codex in an Agent-First World — OpenAI](https://openai.com/index/harness-engineering/)
- [Effective Harnesses for Long-Running Agents — Anthropic](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents)
- [What Is an Agent Harness? — Firecrawl](https://www.firecrawl.dev/blog/what-is-an-agent-harness)
- [What is AI Harness Engineering? — Medium](https://medium.com/be-open/what-is-ai-harness-engineering-your-guide-to-controlling-autonomous-systems-30c9c8d2b489)
- [The Importance of Agent Harness in 2026 — Philipp Schmid](https://www.philschmid.de/agent-harness-2026)
- [2025 Was Agents. 2026 Is Agent Harnesses. — Aakash Gupta](https://aakashgupta.medium.com/2025-was-agents-2026-is-agent-harnesses-heres-why-that-changes-everything-073e9877655e)
- [The State of AI Agent Frameworks — Roberto Infante](https://medium.com/@roberto.g.infante/the-state-of-ai-agent-frameworks-comparing-langgraph-openai-agent-sdk-google-adk-and-aws-d3e52a497720)
- [Everything Claude Code: The Agent Harness — Big Hat Group](https://www.bighatgroup.com/blog/everything-claude-code-ai-agent-harness-guide/)
- [Claude Agent SDK Overview — Anthropic](https://platform.claude.com/docs/en/agent-sdk/overview)
