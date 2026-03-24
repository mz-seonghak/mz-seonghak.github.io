---
layout: post
title: "Strands Agents SDK — AWS가 공개한 오픈소스 AI Agent 프레임워크 깊이 파헤치기"
date: 2026-03-24
tags: [strands-agents, aws, ai-agent, multi-agent, swarm, mcp, open-source, python, typescript]
---

AWS가 **Strands Agents**라는 오픈소스 AI Agent SDK를 공개했습니다. Amazon Q Developer, AWS Glue, VPC Reachability Analyzer 등 AWS 내부 프로덕션 환경에서 이미 사용 중인 프레임워크를 외부에 공개한 것입니다.

기존 에이전트 프레임워크들이 복잡한 오케스트레이션 로직을 요구했다면, Strands는 **모델 주도(model-driven) 접근 방식**으로 몇 줄의 코드만으로 프로덕션 수준의 AI Agent를 구축할 수 있게 해줍니다.

이 글에서는 Strands Agents SDK의 핵심 개념, 멀티 에이전트 패턴, 프로덕션 배포 아키텍처까지 전반적으로 살펴보겠습니다.

---

## 1. Strands Agents는 왜 만들어졌나

AWS의 Amazon Q Developer 팀은 2023년 초, 원조 [ReAct 논문](https://arxiv.org/pdf/2210.03629)이 발표될 무렵부터 AI Agent를 구축해왔습니다. 당시에는 LLM이 에이전트처럼 동작하도록 훈련되지 않았기 때문에, 도구 사용을 위한 복잡한 프롬프트 지시, 응답 파서, 오케스트레이션 로직이 필요했고 프로토타입에서 프로덕션까지 **수개월**이 걸렸습니다.

그러나 최신 LLM들은 네이티브로 도구 사용(tool use)과 추론(reasoning) 기능을 지원합니다. 기존 프레임워크의 복잡한 오케스트레이션이 오히려 최신 모델의 능력을 제한하는 상황이 된 것입니다.

Strands Agents는 이 복잡성을 제거하기 위해 탄생했습니다. 결과적으로 Q Developer 팀은 새로운 에이전트를 **수개월이 아닌 수일~수주** 만에 프로덕션에 배포할 수 있게 되었습니다.

---

## 2. 핵심 개념: Model + Tools + Prompt

Strands에서 에이전트는 세 가지 구성 요소로 정의됩니다.

| 구성 요소 | 설명 |
|-----------|------|
| **Model** | Amazon Bedrock, Anthropic API, Llama API, Ollama, OpenAI(LiteLLM 경유) 등 어떤 LLM이든 사용 가능 |
| **Tools** | MCP 서버, 20개 이상의 내장 도구, `@tool` 데코레이터로 Python 함수를 도구화 |
| **Prompt** | 에이전트의 작업을 정의하는 자연어 프롬프트 및 시스템 프롬프트 |

DNA의 두 가닥(strands)처럼 **모델과 도구**를 연결하는 것이 핵심 철학입니다. 모델이 계획을 세우고, 도구를 선택하고, 결과를 반영하는 모든 과정을 모델의 추론 능력에 위임합니다.

### Agentic Loop

Strands의 에이전틱 루프는 다음과 같이 동작합니다:

1. 프롬프트와 에이전트 컨텍스트, 도구 설명을 LLM에 전달
2. LLM이 자연어 응답, 단계 계획, 이전 단계 반영, 도구 선택 중 하나 이상을 수행
3. 도구가 선택되면 Strands가 도구를 실행하고 결과를 LLM에 반환
4. 작업이 완료될 때까지 반복

---

## 3. 빠른 시작 예제

설치는 한 줄로 끝납니다:

```bash
pip install strands-agents strands-agents-tools
```

가장 기본적인 에이전트 코드입니다:

```python
from strands import Agent

agent = Agent(system_prompt="You are a helpful assistant.")
agent("서울의 오늘 날씨를 알려줘")
```

MCP 서버와 내장 도구를 결합한 좀 더 실용적인 예제를 살펴보겠습니다:

```python
from strands import Agent
from strands.tools.mcp import MCPClient
from strands_tools import http_request
from mcp import stdio_client, StdioServerParameters

NAMING_SYSTEM_PROMPT = """
You are an assistant that helps to name open source projects.

When providing open source project name suggestions, always provide
one or more available domain names and one or more available GitHub
organization names that could be used for the project.

Before providing your suggestions, use your tools to validate
that the domain names are not already registered and that the GitHub
organization names are not already used.
"""

domain_name_tools = MCPClient(lambda: stdio_client(
    StdioServerParameters(command="uvx", args=["fastdomaincheck-mcp-server"])
))

github_tools = [http_request]

with domain_name_tools:
    tools = domain_name_tools.list_tools_sync() + github_tools
    naming_agent = Agent(
        system_prompt=NAMING_SYSTEM_PROMPT,
        tools=tools
    )
    naming_agent("I need to name an open source project for building AI agents.")
```

MCP 서버를 `MCPClient`로 감싸서 도구로 등록하고, `strands_tools`의 내장 도구와 결합하는 것이 전부입니다.

### @tool 데코레이터

Python 함수를 도구로 만드는 것도 간단합니다:

```python
from strands import tool

@tool
def get_weather(city: str) -> str:
    """주어진 도시의 현재 날씨를 반환합니다."""
    return f"{city}의 현재 날씨: 맑음, 18°C"
```

---

## 4. 내장 도구 하이라이트

Strands는 20개 이상의 내장 도구를 제공합니다. 그중 주목할 만한 것들입니다:

### Retrieve Tool

Amazon Bedrock Knowledge Bases를 활용한 시맨틱 검색 도구입니다. 문서 검색 외에도 **도구 자체를 시맨틱 검색으로 찾는** 패턴을 지원합니다. AWS 내부의 한 에이전트는 **6,000개 이상의 도구** 중에서 현재 작업에 관련된 도구만 시맨틱 검색으로 선별해서 모델에 제공합니다.

### Thinking Tool

모델이 여러 사이클에 걸쳐 깊은 분석적 사고를 하도록 유도하는 도구입니다. 모델 주도 접근에서 사고(thinking)를 도구로 모델링하면, 모델이 언제 깊은 분석이 필요한지 스스로 판단할 수 있습니다.

### Multi-Agent Tools

Workflow, Graph, Swarm 도구를 통해 복잡한 작업을 여러 에이전트가 협력하여 처리합니다. 서브 에이전트와 멀티 에이전트 협업을 도구로 모델링하여, 모델이 어떤 작업에 어떤 협업 패턴이 필요한지 추론할 수 있습니다.

---

## 5. 멀티 에이전트 패턴

Strands는 세 가지 멀티 에이전트 패턴을 지원합니다.

### 5.1 Swarm 패턴

여러 에이전트가 **자율적으로 협업**하는 패턴입니다. 중앙 제어 없이 에이전트들이 공유 컨텍스트와 작업 메모리를 기반으로 자체 조직화됩니다.

```python
from strands import Agent
from strands.multiagent import Swarm

researcher = Agent(name="researcher", system_prompt="리서치 전문가입니다...")
coder = Agent(name="coder", system_prompt="코딩 전문가입니다...")
reviewer = Agent(name="reviewer", system_prompt="코드 리뷰 전문가입니다...")
architect = Agent(name="architect", system_prompt="시스템 아키텍처 전문가입니다...")

swarm = Swarm(
    [coder, researcher, reviewer, architect],
    entry_point=researcher,
    max_handoffs=20,
    max_iterations=20,
    execution_timeout=900.0,
    node_timeout=300.0,
)

result = swarm("TODO 앱을 위한 간단한 REST API를 설계하고 구현하세요")
```

Swarm의 핵심 동작:

- 각 에이전트는 전체 작업 컨텍스트에 접근 가능
- 어떤 에이전트가 작업했는지 히스토리 확인 가능
- 다른 에이전트가 기여한 공유 지식에 접근 가능
- 다른 전문성이 필요하면 `handoff_to_agent` 도구로 제어권 이전

안전장치로 **최대 핸드오프 횟수**, **실행 타임아웃**, **반복 핸드오프 감지**(핑퐁 방지) 등이 내장되어 있습니다.

### 5.2 Graph 패턴

에이전트들을 **방향 그래프의 노드**로 배치하고, 의존성에 따라 결정적(deterministic)으로 실행하는 패턴입니다.

- DAG(비순환) 및 순환 토폴로지 지원
- 조건부 엣지 순회로 동적 워크플로우 구성
- A2A 프로토콜을 통한 원격 에이전트 지원

```python
from strands import Agent
from strands.multiagent.graph import GraphBuilder

researcher = Agent(name="researcher", system_prompt="...")
writer = Agent(name="writer", system_prompt="...")
editor = Agent(name="editor", system_prompt="...")

graph = (
    GraphBuilder()
    .add_node("research", researcher)
    .add_node("write", writer)
    .add_node("edit", editor)
    .add_edge("research", "write")
    .add_edge("write", "edit")
    .set_entry_point("research")
    .build()
)

result = graph("양자 컴퓨팅의 최신 동향을 분석하세요")
```

### 5.3 Agent-to-Agent (A2A) 프로토콜

분산 환경에서 에이전트 간 통신을 위한 프로토콜입니다. `StrandsA2AExecutor`가 Strands Agent를 A2A 프로토콜에 적응시켜, 에이전트 요청 실행과 스트리밍 이벤트 변환을 처리합니다.

---

## 6. 프로덕션 배포 아키텍처

Strands는 프로덕션 운영을 핵심 설계 원칙으로 삼고 있습니다. 네 가지 배포 아키텍처를 지원합니다.

### 아키텍처 1: 로컬 실행

에이전트가 사용자 환경에서 완전히 로컬로 실행됩니다. CLI 기반 AI 어시스턴트에 적합합니다.

### 아키텍처 2: API 뒤의 모놀리스

에이전트와 도구가 하나의 환경에서 API 뒤에 배포됩니다. AWS Lambda, AWS Fargate, Amazon EC2에 배포하는 레퍼런스 구현이 제공됩니다.

### 아키텍처 3: 에이전트/도구 분리

에이전틱 루프와 도구 실행을 별도 환경에서 운영합니다. 예를 들어 에이전트는 Fargate 컨테이너에서, 도구는 Lambda 함수에서 실행할 수 있습니다.

### 아키텍처 4: Return-of-Control

클라이언트가 도구 실행을 담당합니다. 백엔드 환경의 도구와 클라이언트 로컬 도구를 혼합하여 사용할 수 있습니다.

### 관측성(Observability)

Strands는 **OpenTelemetry(OTEL)**를 사용하여 에이전트 실행 경로와 메트릭을 수집합니다. 분산 추적(distributed tracing)으로 아키텍처의 여러 컴포넌트를 거치는 요청을 추적하여 에이전트 세션의 전체 그림을 파악할 수 있습니다.

---

## 7. Strands Labs — 실험적 프로젝트

2026년 3월, AWS는 **[Strands Labs](https://github.com/strands-labs)**라는 실험적 프로젝트 조직을 공개했습니다.

### Robots

AI 에이전트와 물리적 하드웨어를 연결합니다. NVIDIA GR00T 모델과 통합하여 SO-101 로봇 팔을 제어하는 예제를 시연했습니다. HuggingFace의 LeRobot 프레임워크와도 통합됩니다.

### Robots Sim

물리적 하드웨어 없이 시뮬레이션 환경에서 로봇 에이전트를 실험할 수 있습니다. Libero 로보틱스 벤치마크 환경을 지원합니다.

### AI Functions

자연어 설명과 Python 검증 조건으로 함수의 동작을 정의하면, Strands 에이전트 루프가 코드를 생성하고 검증하는 새로운 프로그래밍 패러다임입니다.

```python
@ai_function
def parse_log_entries(log_text: str) -> pd.DataFrame:
    """로그 텍스트를 파싱하여 타임스탬프, 레벨, 메시지 컬럼을 가진 DataFrame을 반환합니다."""
    ...
```

검증에 실패하면 자동으로 재시도하여 사양을 만족하는 구현을 생성합니다.

---

## 8. 생태계와 커뮤니티

Strands Agents는 Apache License 2.0 오픈소스 프로젝트입니다.

| 항목 | 상세 |
|------|------|
| Python SDK | [strands-agents/sdk-python](https://github.com/strands-agents/sdk-python) (5,300+ stars) |
| TypeScript SDK | [strands-agents/sdk-typescript](https://github.com/strands-agents/sdk-typescript) (500+ stars) |
| 도구 패키지 | [strands-agents/tools](https://github.com/strands-agents/tools) |
| MCP 서버 | [strands-agents/mcp-server](https://github.com/strands-agents/mcp-server) |
| 라이선스 | Apache License 2.0 |

기여 기업으로 **Accenture, Anthropic, Langfuse, mem0.ai, Meta, PwC, Ragas.io, Tavily** 등이 참여하고 있습니다. Anthropic은 Claude 모델 지원을, Meta는 Llama 모델 지원을 직접 기여했습니다.

실제 도입 사례도 인상적입니다:

- **Eightcap**: 조사 시간 평균 30분 → 45초로 단축, 조사 품질 94% 향상, 운영 비용 500만 달러 절감
- **Smartsheet**: 차세대 AI 기능의 기반으로 채택
- **Swisscom**: 수주 내 PoC 구축, 멀티 에이전트 시스템 확장 중

---

## 9. 다른 프레임워크와의 비교

| 특성 | Strands Agents | LangGraph | CrewAI |
|------|---------------|-----------|--------|
| 접근 방식 | 모델 주도 | 그래프 기반 워크플로우 | 역할 기반 |
| 복잡도 | 낮음 (몇 줄의 코드) | 중간~높음 | 중간 |
| 모델 지원 | Bedrock, Anthropic, Llama, Ollama, LiteLLM | 주로 OpenAI, Anthropic | 주로 OpenAI |
| 도구 통합 | MCP 네이티브 지원 | 커스텀 도구 | 커스텀 도구 |
| AWS 통합 | 네이티브 (Bedrock AgentCore, Lambda, EKS 등) | 제한적 | 제한적 |
| 프로덕션 배포 | 4가지 레퍼런스 아키텍처 제공 | LangServe | 제한적 |
| 관측성 | OpenTelemetry 네이티브 | LangSmith | 제한적 |
| 멀티 에이전트 | Swarm, Graph, A2A | Graph | Crew (역할 기반) |

Strands의 차별점은 **"모델이 충분히 똑똑하다"**는 전제에서 출발한다는 것입니다. 복잡한 오케스트레이션 코드 대신 모델의 추론 능력에 의존하므로, 모델 성능이 향상되면 에이전트 성능도 자연스럽게 따라갑니다.

---

## 10. 정리

Strands Agents SDK는 AI Agent 개발의 복잡성을 줄이면서도 프로덕션 수준의 안정성을 제공하는 것이 핵심 가치입니다.

- **간결함**: 프롬프트와 도구 목록만으로 에이전트 정의
- **유연함**: 모델, 도구, 배포 환경 모두 교체 가능
- **확장성**: 단일 에이전트에서 Swarm, Graph, A2A 멀티 에이전트까지
- **프로덕션 준비**: AWS 내부에서 검증된 배포 패턴과 관측성
- **개방성**: Apache 2.0 라이선스, MCP 네이티브 지원, 다양한 기업 기여

AWS가 내부에서 수년간 갈고닦은 에이전트 프레임워크를 공개한 만큼, 특히 AWS 생태계에서 AI Agent를 구축하려는 팀이라면 가장 먼저 검토해볼 선택지입니다.

---

## 참고 링크

- [Strands Agents 공식 사이트](https://strandsagents.com/)
- [AWS Open Source Blog - Introducing Strands Agents](https://aws.amazon.com/blogs/opensource/introducing-strands-agents-an-open-source-ai-agents-sdk)
- [GitHub - strands-agents](https://github.com/strands-agents)
- [GitHub - Strands Labs](https://github.com/strands-labs)
- [InfoQ - AWS Launches Strands Labs](https://www.infoq.com/news/2026/03/aws-strands-agents/)
