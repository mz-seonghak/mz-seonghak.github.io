---
layout: post
title: "Google ADK 멀티에이전트 시스템: 컨텍스트 공유와 에스컬레이션 완전 가이드"
date: 2026-03-24
tags: [adk, google, multi-agent, context, escalation, session, state, tool]
---

Google의 **Agent Development Kit(ADK)**으로 멀티에이전트 시스템을 구축할 때, 가장 핵심적인 질문 중 하나는 **"에이전트들이 어떻게 정보를 공유하고, 처리할 수 없는 작업을 다른 에이전트에게 넘기는가?"**입니다.

이 글에서는 ADK에서 제공하는 컨텍스트 공유 메커니즘과 에스컬레이션 패턴을 네 가지 축으로 정리합니다.

1. **Agent ↔ Agent** 간 컨텍스트 공유
2. **Agent ↔ SubAgent** 간 컨텍스트 공유
3. **Agent ↔ Tool** 간 컨텍스트 공유
4. **Escalation** — 다른 에이전트나 상위 에이전트로 제어를 되돌리는 방법

---

## 1. ADK의 컨텍스트 관리 구조

ADK는 대화형 컨텍스트를 **세 가지 계층**으로 관리합니다.

```
Session (대화 스레드)
├── Events[] (메시지/액션 이력)
├── State (session.state) ← 현재 대화의 임시 데이터
│   ├── 기본 키 (세션 스코프)
│   ├── user: 접두사 (사용자 스코프)
│   ├── app: 접두사 (앱 전역 스코프)
│   └── temp: 접두사 (단일 호출 스코프)
└── Memory (교차 세션 장기 기억)
```

| 구성 요소 | 역할 | 지속성 |
|-----------|------|--------|
| **Session** | 단일 대화 스레드. 사용자와 에이전트 간 상호작용의 이벤트 시퀀스 포함 | SessionService에 따라 결정 |
| **State** | 세션 내 키-값 쌍으로 된 동적 데이터 저장소 | 접두사(prefix)에 따라 스코프 결정 |
| **Memory** | 여러 세션에 걸친 검색 가능한 장기 지식 저장소 | MemoryService가 관리 |

### State의 네 가지 스코프 접두사

ADK State의 핵심은 **접두사(prefix)**로 스코프를 구분한다는 점입니다.

```python
# 세션 스코프 — 현재 세션에서만 유효
session.state['current_step'] = 'analysis'

# 사용자 스코프 — 같은 사용자의 모든 세션에서 공유
session.state['user:preferred_language'] = 'ko'

# 앱 스코프 — 모든 사용자와 세션에서 공유
session.state['app:global_config'] = 'v2'

# 임시 스코프 — 현재 호출(invocation)에서만 유효, 호출 완료 후 폐기
session.state['temp:intermediate_result'] = {'score': 0.95}
```

특히 `temp:` 접두사는 **부모 에이전트가 서브에이전트를 호출할 때 같은 InvocationContext를 전달**하기 때문에, 동일 호출 체인 내에서 에이전트 간 데이터를 임시로 주고받는 데 유용합니다.

---

## 2. Agent ↔ Agent 간 컨텍스트 공유

같은 시스템 내 에이전트들이 서로 데이터를 교환하는 방법은 크게 세 가지입니다.

### 2-1. Shared Session State (공유 세션 상태)

가장 기본적이고 널리 사용되는 방식입니다. 같은 Session을 공유하는 에이전트들은 `session.state`를 통해 데이터를 읽고 쓸 수 있습니다.

```python
from google.adk.agents import LlmAgent, SequentialAgent

agent_a = LlmAgent(
    name="AgentA",
    model="gemini-2.0-flash",
    instruction="프랑스의 수도를 찾아주세요.",
    output_key="capital_city"  # 결과를 state['capital_city']에 저장
)

agent_b = LlmAgent(
    name="AgentB",
    model="gemini-2.0-flash",
    instruction="{capital_city}에 대해 자세히 알려주세요."
    # {capital_city} → state['capital_city'] 값이 자동 주입됨
)

pipeline = SequentialAgent(
    name="CityInfoPipeline",
    sub_agents=[agent_a, agent_b]
)
```

여기서 핵심은 `output_key` 파라미터입니다. 에이전트의 최종 텍스트 응답이 자동으로 지정된 State 키에 저장되어, 후속 에이전트가 `{key}` 템플릿 구문으로 바로 참조할 수 있습니다.

### 2-2. LLM-Driven Delegation (transfer_to_agent)

LLM이 상황을 판단하여 다른 에이전트에게 **동적으로 제어를 넘기는** 방식입니다.

```python
from google.adk.agents import LlmAgent

billing_agent = LlmAgent(
    name="Billing",
    model="gemini-2.0-flash",
    description="결제 관련 문의를 처리합니다."
)

support_agent = LlmAgent(
    name="Support",
    model="gemini-2.0-flash",
    description="기술 지원 요청을 처리합니다."
)

coordinator = LlmAgent(
    name="HelpDesk",
    model="gemini-2.0-flash",
    instruction="결제 문제는 Billing에게, 기술 문제는 Support에게 전달하세요.",
    sub_agents=[billing_agent, support_agent]
)
```

사용자가 "결제가 안 돼요"라고 말하면, Coordinator의 LLM이 자동으로 다음과 같은 함수 호출을 생성합니다.

```
transfer_to_agent(agent_name='Billing')
```

ADK 프레임워크의 `AutoFlow`가 이 호출을 가로채서 `root_agent.find_agent()`로 대상 에이전트를 찾고, `InvocationContext`를 갱신하여 실행 초점을 전환합니다.

### 2-3. AgentTool을 통한 명시적 호출

다른 에이전트를 **도구(Tool)처럼 감싸서** 호출하는 방식입니다. `transfer_to_agent`가 제어 자체를 넘기는 것과 달리, `AgentTool`은 현재 에이전트의 흐름 안에서 다른 에이전트를 실행하고 결과를 받아옵니다.

```python
from google.adk.agents import LlmAgent
from google.adk.tools import agent_tool

summarizer = LlmAgent(
    name="Summarizer",
    model="gemini-2.0-flash",
    description="텍스트를 요약합니다."
)

research_agent = LlmAgent(
    name="Researcher",
    model="gemini-2.0-flash",
    instruction="주제를 조사하고, Summarizer 도구를 사용해 결과를 요약하세요.",
    tools=[agent_tool.AgentTool(agent=summarizer)]
)
```

| 방식 | 제어 흐름 | 사용 시나리오 |
|------|----------|-------------|
| `transfer_to_agent` | 제어권 자체가 대상 에이전트로 이동 | 전문 에이전트에게 대화를 완전히 위임 |
| `AgentTool` | 현재 에이전트 안에서 대상 에이전트를 실행 후 결과 수신 | 부분 작업을 도구처럼 위임하고 결과를 조합 |

---

## 3. Agent ↔ SubAgent 간 컨텍스트 공유

ADK에서 에이전트 계층은 트리 구조로 구성됩니다. 부모 에이전트가 서브에이전트를 호출할 때 **같은 `InvocationContext`를 전달**하므로, 여러 가지 메커니즘으로 컨텍스트를 공유할 수 있습니다.

### 3-1. InvocationContext 공유

```
Coordinator (부모)
├── InvocationContext ──── 공유 ────→ SubAgent A
│   ├── session
│   ├── state (temp: 포함)        ──→ SubAgent B
│   └── services
```

부모 에이전트가 `SequentialAgent`나 `ParallelAgent`로 서브에이전트를 실행하면, 같은 `InvocationContext`가 전달됩니다. 이는 곧 **동일한 `temp:` 상태를 공유**한다는 의미입니다.

```python
from google.adk.agents import SequentialAgent, LlmAgent

# Step1이 temp: 상태에 데이터를 저장
step1 = LlmAgent(
    name="Analyzer",
    model="gemini-2.0-flash",
    instruction="데이터를 분석하세요.",
    output_key="temp:analysis_result"
)

# Step2가 같은 temp: 상태에서 데이터를 읽음
step2 = LlmAgent(
    name="Reporter",
    model="gemini-2.0-flash",
    instruction="{temp:analysis_result}를 바탕으로 보고서를 작성하세요."
)

pipeline = SequentialAgent(
    name="AnalysisPipeline",
    sub_agents=[step1, step2]
)
```

### 3-2. Workflow Agent별 컨텍스트 전달 방식

| Workflow Agent | 실행 방식 | 컨텍스트 특성 |
|---------------|----------|-------------|
| **SequentialAgent** | 순차 실행 | 같은 InvocationContext를 순서대로 전달. 이전 에이전트의 State 변경이 다음 에이전트에 즉시 반영 |
| **ParallelAgent** | 병렬 실행 | 각 자식에게 다른 `branch` 경로를 부여하지만, **같은 `session.state`를 공유**. 경합 방지를 위해 서로 다른 키를 사용해야 함 |
| **LoopAgent** | 반복 실행 | 매 반복마다 같은 InvocationContext를 전달. State 변경이 다음 반복에 누적됨 |

### 3-3. Parallel Agent에서의 State 공유 주의점

```python
from google.adk.agents import ParallelAgent, SequentialAgent, LlmAgent

# 병렬 실행 시 서로 다른 output_key를 사용해야 경합 조건을 방지
fetch_weather = LlmAgent(
    name="WeatherFetcher",
    model="gemini-2.0-flash",
    instruction="날씨 정보를 가져오세요.",
    output_key="weather_data"  # 고유한 키
)

fetch_news = LlmAgent(
    name="NewsFetcher",
    model="gemini-2.0-flash",
    instruction="뉴스를 가져오세요.",
    output_key="news_data"  # 고유한 키
)

gather = ParallelAgent(
    name="InfoGatherer",
    sub_agents=[fetch_weather, fetch_news]
)

synthesizer = LlmAgent(
    name="Synthesizer",
    model="gemini-2.0-flash",
    instruction="{weather_data}와 {news_data}를 종합하세요."
)

workflow = SequentialAgent(
    name="GatherAndSynthesize",
    sub_agents=[gather, synthesizer]
)
```

---

## 4. Agent ↔ Tool 간 컨텍스트 공유

도구(Tool)는 에이전트의 실행 환경과 상태에 접근해야 할 때가 많습니다. ADK는 이를 위해 **ToolContext**를 제공합니다.

### 4-1. ToolContext의 구조

`ToolContext`는 `CallbackContext`를 확장한 것으로, 도구 함수 내에서 세션 상태, 아티팩트, 메모리, 인증 등 프레임워크의 다양한 기능에 접근할 수 있게 합니다.

```
ToolContext (도구 함수에 전달)
├── state (읽기/쓰기 가능)
│   ├── session.state['key'] 읽기
│   └── session.state['key'] 쓰기 → EventActions.state_delta로 자동 추적
├── load_artifact() / save_artifact()
├── list_artifacts()
├── search_memory(query)
├── request_credential() / get_auth_response()
├── function_call_id
└── actions (EventActions 직접 접근)
```

### 4-2. 도구에서 State 읽기/쓰기

```python
from google.adk.tools import ToolContext

def search_database(query: str, tool_context: ToolContext) -> dict:
    # State에서 사용자 설정 읽기
    user_lang = tool_context.state.get("user:preferred_language", "en")
    
    # 도구 실행 로직
    results = perform_search(query, language=user_lang)
    
    # State에 결과 저장 (자동으로 EventActions.state_delta에 추적됨)
    tool_context.state["last_search_query"] = query
    tool_context.state["temp:search_result_count"] = len(results)
    
    return {"results": results, "count": len(results)}
```

도구 함수에서 `tool_context.state`를 수정하면, ADK 프레임워크가 자동으로 이 변경을 `EventActions.state_delta`에 포함시킵니다. 수동으로 `EventActions`를 구성할 필요가 없습니다.

### 4-3. CallbackContext를 통한 콜백에서의 State 접근

에이전트의 콜백 함수(`before_agent_callback`, `after_agent_callback` 등)에서도 `CallbackContext`를 통해 동일하게 State에 접근할 수 있습니다.

```python
from google.adk.agents.context import Context
from google.adk.models import LlmRequest
from google.genai import types

def before_model_callback(context: Context, request: LlmRequest):
    call_count = context.state.get("model_calls", 0)
    context.state["model_calls"] = call_count + 1
    
    # 특정 조건에서 모델 호출을 가로채기
    if context.state.get("temp:skip_model"):
        return types.Content(
            parts=[types.Part(text="캐시된 응답을 반환합니다.")]
        )
    
    return None  # 정상적으로 모델 호출 진행
```

### 4-4. 도구에서 메모리와 아티팩트 활용

`ToolContext`는 State 외에도 메모리 검색과 아티팩트 관리 기능을 제공합니다.

```python
from google.adk.tools import ToolContext

async def intelligent_search(query: str, tool_context: ToolContext) -> dict:
    # 과거 세션 메모리에서 관련 정보 검색
    relevant_memories = await tool_context.search_memory(
        f"{query}와 관련된 과거 대화"
    )
    
    # 세션 아티팩트 목록 조회
    artifacts = await tool_context.list_artifacts()
    
    # 특정 아티팩트 로드
    config = await tool_context.load_artifact("search_config.json")
    
    # 결과를 아티팩트로 저장
    await tool_context.save_artifact(
        "search_results.json",
        types.Part(text=json.dumps(results))
    )
    
    return {"results": results, "memory_context": relevant_memories}
```

---

## 5. Escalation — 제어를 상위 또는 다른 에이전트로 되돌리기

Escalation은 서브에이전트가 작업을 완료했거나, 자신이 처리할 수 없는 상황에서 **상위 에이전트나 다른 에이전트에게 제어를 반환**하는 메커니즘입니다.

### 5-1. EventActions.escalate — LoopAgent 탈출

`LoopAgent` 안에서 서브에이전트가 `escalate=True`를 설정한 이벤트를 발생시키면, 루프가 즉시 종료됩니다.

```python
from google.adk.agents import LoopAgent, LlmAgent, BaseAgent
from google.adk.events import Event, EventActions
from google.adk.agents.invocation_context import InvocationContext

class QualityGate(BaseAgent):
    async def _run_async_impl(self, ctx: InvocationContext):
        status = ctx.session.state.get("quality_status", "fail")
        should_stop = (status == "pass")
        yield Event(
            author=self.name,
            actions=EventActions(escalate=should_stop)
        )

code_refiner = LlmAgent(
    name="CodeRefiner",
    model="gemini-2.0-flash",
    instruction="코드를 개선하세요. 현재 코드: {current_code}",
    output_key="current_code"
)

quality_checker = LlmAgent(
    name="QualityChecker",
    model="gemini-2.0-flash",
    instruction="{current_code}의 품질을 평가하세요. 'pass' 또는 'fail'로 답하세요.",
    output_key="quality_status"
)

refinement_loop = LoopAgent(
    name="RefinementLoop",
    max_iterations=5,
    sub_agents=[code_refiner, quality_checker, QualityGate(name="Gate")]
)
```

동작 흐름:

```
[반복 1] CodeRefiner → QualityChecker → Gate(escalate=False) → 계속
[반복 2] CodeRefiner → QualityChecker → Gate(escalate=False) → 계속
[반복 3] CodeRefiner → QualityChecker(pass!) → Gate(escalate=True) → 루프 종료
```

### 5-2. transfer_to_agent — 부모/형제 에이전트로 전환

서브에이전트가 `transfer_to_agent`를 호출하여 부모 에이전트나 형제 에이전트에게 제어를 넘길 수 있습니다. 이는 LLM이 자연어 이해를 기반으로 동적으로 결정합니다.

```python
billing_agent = LlmAgent(
    name="Billing",
    model="gemini-2.0-flash",
    description="결제 관련 문의를 처리합니다.",
    instruction="""결제 관련 질문에 답하세요.
    기술 지원이 필요한 질문이면 Support 에이전트로 전환하세요.
    일반적인 문의면 HelpDesk(부모)로 전환하세요."""
)

support_agent = LlmAgent(
    name="Support",
    model="gemini-2.0-flash",
    description="기술 지원을 제공합니다.",
    instruction="""기술 문제를 해결하세요.
    결제 관련 문의면 Billing 에이전트로 전환하세요."""
)

coordinator = LlmAgent(
    name="HelpDesk",
    model="gemini-2.0-flash",
    instruction="사용자 요청을 분석해 적절한 에이전트에게 전달하세요.",
    sub_agents=[billing_agent, support_agent]
)
```

전환 가능한 범위:

```
HelpDesk (부모)
├── Billing ←→ Support (형제 간 전환 가능)
│   └── Billing → HelpDesk (자식 → 부모 전환 가능)
└── Support → HelpDesk (자식 → 부모 전환 가능)
```

ADK에서는 `disallow_transfer_to_parent`와 `disallow_transfer_to_peers` 옵션으로 전환 범위를 세밀하게 제어할 수 있습니다.

### 5-3. fallback_to_parent — 자동 부모 복귀

ADK에 추가된 `fallback_to_parent` 기능은 서브에이전트가 작업을 완료한 후 **자동으로 부모 에이전트에게 제어를 반환**합니다.

```python
specialist = LlmAgent(
    name="DataAnalyst",
    model="gemini-2.0-flash",
    description="데이터 분석 전문가입니다.",
    instruction="요청된 데이터 분석을 수행하세요.",
    fallback_to_parent=True  # 작업 완료 후 자동으로 부모에게 복귀
)

coordinator = LlmAgent(
    name="ProjectManager",
    model="gemini-2.0-flash",
    instruction="프로젝트 관리를 담당합니다. 데이터 분석이 필요하면 DataAnalyst에게 위임하세요.",
    sub_agents=[specialist]
)
```

`fallback_to_parent=True`가 동작하는 조건:

1. 해당 에이전트가 `LlmAgent` 인스턴스일 것
2. `fallback_to_parent=True`로 설정되어 있을 것
3. 부모 에이전트가 존재할 것
4. 모델 응답에 **명시적 `transfer_to_agent` 호출이 없을** 것

이 네 가지 조건이 모두 충족되면, 에이전트는 실행을 마친 후 자동으로 `transfer_to_agent(parent_name)`을 생성하여 부모에게 돌아갑니다.

### 5-4. Escalation 패턴 비교

| 메커니즘 | 트리거 | 대상 | 주요 사용처 |
|---------|--------|------|-----------|
| `escalate=True` | 커스텀 에이전트가 이벤트에 설정 | LoopAgent → 루프 탈출 | 반복 작업의 종료 조건 |
| `transfer_to_agent` | LLM이 동적으로 판단 | 부모/형제/자식 에이전트 | 대화 라우팅, 작업 위임 |
| `fallback_to_parent` | 자동 (응답 완료 시) | 부모 에이전트 | 단일 작업 위임 후 자동 복귀 |
| `AgentTool` 반환 | 도구 실행 완료 시 | 호출한 에이전트 | 부분 작업 위임 (도구 패턴) |

---

## 6. 실전 종합 예제: 고객 지원 시스템

지금까지 다룬 모든 메커니즘을 결합한 고객 지원 멀티에이전트 시스템 예제입니다.

```python
from google.adk.agents import (
    LlmAgent, SequentialAgent, ParallelAgent, LoopAgent, BaseAgent
)
from google.adk.tools import agent_tool, ToolContext
from google.adk.events import Event, EventActions


# === 도구 정의: ToolContext를 통한 상태 접근 ===

def lookup_customer(customer_id: str, tool_context: ToolContext) -> dict:
    """고객 정보 조회 도구 — State를 통해 결과를 공유"""
    customer = db.get_customer(customer_id)
    tool_context.state["user:customer_tier"] = customer["tier"]
    tool_context.state["temp:customer_name"] = customer["name"]
    return customer

def check_order_status(order_id: str, tool_context: ToolContext) -> dict:
    """주문 상태 확인 — 메모리 검색으로 과거 맥락 활용"""
    past_context = tool_context.search_memory(f"주문 {order_id} 관련 이력")
    status = db.get_order(order_id)
    return {"status": status, "history": past_context}


# === 전문 서브에이전트 ===

billing_specialist = LlmAgent(
    name="BillingSpecialist",
    model="gemini-2.0-flash",
    description="결제, 환불, 청구서 관련 문제를 처리합니다.",
    instruction="""결제 관련 문제를 해결하세요.
    고객 등급은 {user:customer_tier}입니다.
    기술 문제라면 TechSupport로 전환하세요.
    해결 완료되면 요약을 작성하세요.""",
    output_key="resolution_summary",
    fallback_to_parent=True  # 완료 후 자동 복귀
)

tech_support = LlmAgent(
    name="TechSupport",
    model="gemini-2.0-flash",
    description="기술 지원과 계정 접근 문제를 처리합니다.",
    instruction="""기술 문제를 해결하세요.
    결제 문제라면 BillingSpecialist로 전환하세요.""",
    output_key="resolution_summary",
    fallback_to_parent=True
)


# === 품질 검증 루프 ===

class ResolutionValidator(BaseAgent):
    """해결 결과 검증 — escalate로 루프 탈출"""
    async def _run_async_impl(self, ctx):
        score = ctx.session.state.get("satisfaction_score", 0)
        yield Event(
            author=self.name,
            actions=EventActions(escalate=(score >= 4))
        )

satisfaction_check = LlmAgent(
    name="SatisfactionChecker",
    model="gemini-2.0-flash",
    instruction="""고객 만족도를 1-5점으로 평가하세요.
    해결 내용: {resolution_summary}""",
    output_key="satisfaction_score"
)

quality_loop = LoopAgent(
    name="QualityAssurance",
    max_iterations=3,
    sub_agents=[satisfaction_check, ResolutionValidator(name="Validator")]
)


# === 루트 에이전트: 전체 오케스트레이션 ===

root_agent = LlmAgent(
    name="CustomerServiceHub",
    model="gemini-2.0-flash",
    instruction="""고객 지원 허브입니다.
    1. 먼저 고객 정보를 조회하세요.
    2. 문제 유형에 따라 적절한 전문가에게 전달하세요.
    3. 해결 후 품질을 검증하세요.""",
    tools=[lookup_customer, check_order_status],
    sub_agents=[billing_specialist, tech_support]
)
```

이 시스템의 데이터 흐름:

```
사용자: "주문 #1234 환불 요청합니다"
    │
    ▼
[CustomerServiceHub]
    ├── lookup_customer() → tool_context.state에 고객 정보 저장
    ├── transfer_to_agent('BillingSpecialist')
    │
    ▼
[BillingSpecialist]
    ├── {user:customer_tier}로 고객 등급 참조 (State 공유)
    ├── 환불 처리
    ├── output_key="resolution_summary"로 결과 저장
    └── fallback_to_parent → CustomerServiceHub로 자동 복귀
    │
    ▼
[CustomerServiceHub]
    └── resolution_summary를 확인하고 최종 응답
```

---

## 7. 컨텍스트 공유 전략 요약

### 메커니즘별 비교표

| 메커니즘 | 방향 | 데이터 유형 | 지속성 | 사용 난이도 |
|---------|------|-----------|--------|-----------|
| `session.state` (기본) | 양방향 | 직렬화 가능한 모든 타입 | 세션 내 | 낮음 |
| `session.state` (`user:`) | 양방향 | 직렬화 가능한 모든 타입 | 사용자 전체 세션 | 낮음 |
| `session.state` (`temp:`) | 양방향 | 직렬화 가능한 모든 타입 | 단일 호출 내 | 낮음 |
| `output_key` | 단방향 (쓰기) | 텍스트 | 세션 내 | 매우 낮음 |
| `{key}` 템플릿 | 단방향 (읽기) | 문자열 변환 가능 | - | 매우 낮음 |
| `ToolContext.state` | 양방향 | 직렬화 가능한 모든 타입 | State 키에 따름 | 낮음 |
| `CallbackContext.state` | 양방향 | 직렬화 가능한 모든 타입 | State 키에 따름 | 낮음 |
| `AgentTool` 반환값 | 단방향 (결과) | 모델 응답 | - | 중간 |
| `search_memory()` | 단방향 (읽기) | 검색 결과 | 장기 | 중간 |
| `Artifact` | 양방향 | 파일/바이너리 | 세션 내 | 중간 |

### 설계 원칙

1. **State 키 네이밍 컨벤션을 정하세요.** 에이전트 간 공유하는 키는 문서화하고, 접두사로 스코프를 명확히 하세요.
2. **ParallelAgent에서는 고유 키를 사용하세요.** 병렬 실행 시 같은 키에 쓰면 경합 조건이 발생합니다.
3. **`temp:`는 호출 체인 내에서만 사용하세요.** 다음 사용자 입력까지 데이터를 유지하려면 기본 키나 `user:` 접두사를 쓰세요.
4. **`fallback_to_parent`로 자동 복귀를 보장하세요.** 단일 작업을 위임하는 서브에이전트에 설정하면, 제어 흐름이 예측 가능해집니다.
5. **`ToolContext`에서 State 변경을 추적하세요.** `session.state`를 직접 수정하지 말고, 항상 Context 객체를 통해 수정해야 변경이 올바르게 추적됩니다.

---

## 참고 자료

- [ADK 공식 문서 - Multi-Agent Systems](https://google.github.io/adk-docs/agents/multi-agents/)
- [ADK 공식 문서 - Context](https://google.github.io/adk-docs/context/)
- [ADK 공식 문서 - State](https://google.github.io/adk-docs/sessions/state/)
- [ADK 공식 문서 - Sessions](https://google.github.io/adk-docs/sessions/)
- [ADK 공식 문서 - Custom Tools](https://google.github.io/adk-docs/tools-custom/)
- [Google Cloud Blog - Agent state and memory with ADK](https://cloud.google.com/blog/topics/developers-practitioners/remember-this-agent-state-and-memory-with-adk)
- [Google Developers Blog - Developer's guide to multi-agent patterns in ADK](https://developers.googleblog.com/developers-guide-to-multi-agent-patterns-in-adk/)
- [ADK GitHub - fallback_to_parent PR #2253](https://github.com/google/adk-python/pull/2253)
