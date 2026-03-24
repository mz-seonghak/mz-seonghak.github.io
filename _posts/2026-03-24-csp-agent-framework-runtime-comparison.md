---
layout: post
title: "CSP별 AI Agent 프레임워크와 런타임 비교: 특화 기능, 오픈소스 대안, Lock-in 분석"
date: 2026-03-24
tags: [agent-engine, bedrock-agentcore, azure-ai-foundry, adk, strands-agents, semantic-kernel, langgraph, open-source, lock-in, multi-cloud]
---

AI Agent를 프로덕션에 배포하려면 두 가지 축을 결정해야 합니다. **에이전트 프레임워크**(개발)와 **매니지드 런타임**(배포·운영)입니다. 3대 CSP(AWS, Azure, GCP)는 각각 이 두 축에 대해 자체 솔루션을 제공하면서 동시에 오픈소스 프레임워크도 지원하는 전략을 취하고 있습니다.

이 글에서는 각 CSP의 에이전트 스택을 **프레임워크 ↔ 런타임** 두 축으로 분리하여 비교하고, 오픈소스 대안과 벤더 Lock-in 리스크를 체계적으로 정리합니다.

---

## 1. 전체 구조 한눈에 보기

| 축 | AWS | Azure | GCP |
|-----|-----|-------|-----|
| **자체 프레임워크** | Strands Agents SDK | Microsoft Agent Framework (Semantic Kernel + AutoGen) | Agent Development Kit (ADK) |
| **프레임워크 라이선스** | Apache 2.0 | MIT | Apache 2.0 |
| **매니지드 런타임** | Bedrock AgentCore | Azure AI Foundry Agent Service | Vertex AI Agent Engine |
| **런타임 특화 기능** | Runtime, Memory, Gateway, Identity, Browser, Code Interpreter, Observability, Evaluations, Policy | 세션/메모리 관리, Bing/Azure AI Search 통합, REST API/Function Apps 자동 호출, Copilot 통합 | 세션 관리, 메모리(단기/장기), VPC-SC, IAM, CMEK, 자동 스케일링 |
| **지원 외부 프레임워크** | LangGraph, CrewAI, LlamaIndex, Google ADK, OpenAI Agents SDK | OpenAI, Anthropic, AWS Bedrock, Ollama 등 | LangGraph, LangChain, CrewAI 등 |

핵심 관찰: 3사 모두 **자체 프레임워크는 오픈소스로 공개**하면서, **매니지드 런타임에서 수익을 창출**하는 동일한 전략을 취하고 있습니다. 프레임워크 레벨에서는 Lock-in이 적지만, 런타임 레벨에서 종속성이 발생합니다.

---

## 2. AWS: Bedrock AgentCore + Strands Agents

### 2.1 Strands Agents SDK (오픈소스 프레임워크)

AWS가 내부적으로 Amazon Q Developer, AWS Glue, VPC Reachability Analyzer 등에서 사용하던 에이전트 프레임워크를 Apache 2.0으로 공개한 것입니다.

**핵심 철학 — "모델 주도(Model-Driven)"**

기존 프레임워크들이 개발자가 오케스트레이션 로직을 명시적으로 작성하도록 요구했다면, Strands는 최신 LLM의 추론 능력에 의존하여 몇 줄의 코드만으로 에이전트를 구성합니다.

```python
from strands import Agent

agent = Agent(system_prompt="You are a helpful assistant.")
agent("서울의 오늘 날씨를 알려줘")
```

**주요 특징:**

- Python/TypeScript 듀얼 SDK
- MCP(Model Context Protocol) 네이티브 지원
- Swarm, Graph, A2A 세 가지 멀티에이전트 패턴
- OpenTelemetry 기반 관측성
- 20+ 내장 도구 (Retrieve, Thinking, Shell, HTTP 등)

### 2.2 Amazon Bedrock AgentCore (매니지드 런타임)

2025년 12월 GA된 AgentCore는 **모듈러 아키텍처**가 특징입니다. 9개 서비스를 독립적으로 또는 조합하여 사용할 수 있습니다.

| 서비스 | 설명 |
|--------|------|
| **Runtime** | 서버리스 배포, 세션별 microVM 격리, 최대 8시간 비동기 처리, 100MB 멀티모달 페이로드 |
| **Memory** | 단기 메모리(멀티턴) + 장기 메모리(세션 간 영속), 에이전트 간 메모리 공유 |
| **Gateway** | API/Lambda/MCP 서버를 MCP 호환 도구로 변환, Salesforce·Zoom·Jira·Slack 통합 |
| **Identity** | Okta, Microsoft Entra ID, Cognito, Auth0 등 기존 IdP 연동 |
| **Code Interpreter** | Python, JavaScript, TypeScript 샌드박스 실행 환경 |
| **Browser** | 클라우드 기반 브라우저 자동화 (Playwright, BrowserUse 호환) |
| **Observability** | OpenTelemetry 호환 트레이싱, 디버깅, 모니터링 |
| **Evaluations** | 에이전트/도구 품질 자동 평가, CloudWatch 통합 |
| **Policy** | Cedar 정책 언어로 도구 호출 전 세밀한 접근 제어 |

**차별점:**

- **프레임워크 무관**: LangGraph, CrewAI, LlamaIndex, Google ADK, OpenAI Agents SDK, Strands 모두 지원
- **모델 무관**: OpenAI, Gemini, Claude, Nova, Llama, Mistral 등 자유롭게 선택
- **과금**: 실제 리소스 소비 기반, I/O 대기 시간은 무료
- **세션 격리**: 각 세션이 전용 microVM에서 실행되어 CPU/메모리/파일시스템 완전 격리

### 2.3 Lock-in 분석

| 항목 | Lock-in 수준 | 설명 |
|------|-------------|------|
| Strands Agents SDK | ❌ 낮음 | Apache 2.0, 어디서든 실행 가능 |
| Bedrock AgentCore Runtime | ⚠️ 중간~높음 | AWS 전용 서비스, microVM 세션 모델은 타 CSP에 없음 |
| AgentCore Memory | ⚠️ 중간 | API는 표준적이나 구현은 AWS 종속 |
| AgentCore Gateway | ⚠️ 중간 | MCP 표준 기반이므로 도구 정의 자체는 이식 가능 |
| AgentCore Identity | 🔴 높음 | AWS IAM/Cognito 깊은 통합 |

**탈출 전략**: Strands SDK + Docker/K8s 자체 배포. Strands는 4가지 배포 아키텍처(로컬, API 모놀리스, 에이전트/도구 분리, Return-of-Control)를 공식 지원합니다.

---

## 3. Azure: AI Foundry Agent Service + Microsoft Agent Framework

### 3.1 Microsoft Agent Framework (오픈소스 프레임워크)

Semantic Kernel과 AutoGen을 통합한 차세대 프레임워크로, 2026년 RC(Release Candidate)에 도달했습니다.

**핵심 특징:**

- .NET과 Python 지원
- 그래프 기반 워크플로 (순차, 동시, 핸드오프, 그룹 채팅)
- MCP, A2A, OpenAPI 등 오픈 표준 지원
- 스트리밍, 체크포인팅, Human-in-the-Loop
- MIT 라이선스

**에이전트 타입이 풍부합니다:**

| 에이전트 타입 | 설명 |
|--------------|------|
| ChatCompletionAgent | 범용 대화형 에이전트 |
| OpenAIAssistantAgent | OpenAI Assistants API 기반 |
| AzureAIAgent | Azure AI Foundry 통합 에이전트 |
| OpenAIResponsesAgent | OpenAI Responses API 기반 |
| CopilotStudioAgent | Microsoft Copilot Studio 연동 |

**다중 모델 공급자 지원**: Azure OpenAI, OpenAI, GitHub Copilot, Anthropic Claude, AWS Bedrock, Ollama 등과 호환됩니다. 이 점에서 Semantic Kernel은 특정 CSP에 종속되지 않는 유연성을 가집니다.

### 3.2 Azure AI Foundry Agent Service (매니지드 런타임)

10,000개 이상의 고객사가 GA 이후 사용 중인 매니지드 서비스입니다.

**메모리 관리가 세분화되어 있습니다:**

- **단기 메모리**: 현재 세션 대화를 추적
- **장기 메모리**: 세션 간 지속되는 영속 메모리, 3단계 프로세스(추출 → 통합 → 검색)로 운영
- **MemorySearchTool**: 네임스페이스로 메모리 격리, 검색 옵션 커스터마이징 가능

**도구 통합:**

- Bing Search, Azure AI Search (지식 검색)
- REST API 자동 호출 (Swagger/OpenAPI 3.0 정의 기반)
- Azure Function Apps 연동
- Azure Logic Apps 통합
- RAG (TextSearchProvider)

**Microsoft 생태계 통합이 최대 강점입니다:**

- Microsoft 365, Teams 원클릭 배포
- Entra ID 기반 거버넌스 및 SSO
- Copilot Studio 연동
- Application Insights 기반 모니터링
- Azure DevOps CI/CD 통합

### 3.3 Lock-in 분석

| 항목 | Lock-in 수준 | 설명 |
|------|-------------|------|
| Semantic Kernel / Agent Framework | ❌ 낮음 | MIT 라이선스, 멀티 모델·멀티 클라우드 지원 |
| Azure AI Foundry Agent Service | 🔴 높음 | Azure 전용, Entra ID·M365·Copilot 깊은 통합 |
| 메모리 서비스 (장기) | 🔴 높음 | Azure 매니지드 서비스, 이식 불가 |
| 도구 통합 (Bing, Azure Functions) | ⚠️ 중간~높음 | Azure 서비스 종속, 단 OpenAPI 기반이므로 스펙은 이식 가능 |
| 모델 접근 | ⚠️ 중간 | Azure OpenAI가 중심, 타 모델은 제한적 |

**탈출 전략**: Semantic Kernel은 OpenAI, Anthropic, Bedrock 등 다양한 모델 백엔드를 지원하므로, 프레임워크 레벨에서는 전환이 용이합니다. 런타임은 Docker/K8s + FastAPI로 직접 구축하되, 메모리 관리와 도구 통합을 재구현해야 합니다.

---

## 4. GCP: Vertex AI Agent Engine + ADK

### 4.1 Agent Development Kit — ADK (오픈소스 프레임워크)

Google이 자체 AI 에이전트 구축에 사용하는 프레임워크를 Apache 2.0으로 공개한 것입니다.

**핵심 특징:**

- SequentialAgent, ParallelAgent, LoopAgent 등 명시적 워크플로 패턴
- 세션 기반 상태 관리 (`session.state`)
- 사용자/앱/세션/임시 4단계 스코프의 상태 관리
- MCP 도구 통합
- A2A 프로토콜 지원

**ADK v1.2.0+ 부터 CLI 단일 명령 배포 지원:**

```bash
# Agent Engine 배포
adk deploy agent_engine \
  --project <project-id> \
  --region us-central1 \
  --staging_bucket <bucket-name>

# Cloud Run 배포 (Agent Engine 없이)
adk deploy cloud_run \
  --project <project-id> \
  --region us-central1
```

### 4.2 Vertex AI Agent Engine (매니지드 런타임)

GCP의 완전 관리형 에이전트 배포 서비스입니다.

**주요 기능:**

- 자동 스케일링
- 세션 관리 (대화 컨텍스트 유지)
- 메모리 서비스: `InMemoryMemoryService` (프로토타이핑) / `VertexAiMemoryBankService` (영속)
- VPC-SC (서비스 경계) 보안
- IAM 기반 접근 제어
- CMEK (고객 관리 암호화 키)
- ADK API 서버 및 웹 UI 제공

**Agent Starter Pack(ASP)**: Terraform/CI/CD 템플릿을 제공하여 새 GCP 프로젝트에서 빠르게 시작할 수 있습니다.

### 4.3 Lock-in 분석

| 항목 | Lock-in 수준 | 설명 |
|------|-------------|------|
| ADK | ❌ 낮음 | Apache 2.0, Cloud Run/GKE/Docker 어디든 배포 가능 |
| Agent Engine | 🔴 높음 | GCP 전용, Vertex AI 종속 |
| MemoryBankService | 🔴 높음 | Agent Engine 의존, 자체 구현 시 InMemory만 기본 제공 |
| VPC-SC / IAM / CMEK | 🔴 높음 | GCP 보안 서비스 고유 |
| Gemini 모델 통합 | ⚠️ 중간 | ADK는 다른 모델도 지원하나 Gemini 최적화 |

**탈출 전략**: `adk deploy cloud_run`으로 Agent Engine 없이 Cloud Run에 직접 배포. 세션 관리는 Firestore, 메모리는 Cloud SQL + Redis로 직접 구현합니다. GCP 완전 이탈 시 Docker/K8s로 어디든 배포 가능합니다.

---

## 5. CSP 매니지드 런타임 기능 상세 비교

### 5.1 세션 및 메모리

| 기능 | Bedrock AgentCore | Azure AI Foundry | Agent Engine |
|------|-------------------|-----------------|-------------|
| 세션 격리 | microVM 격리 (CPU/메모리/FS) | 세션 기반 관리 | 세션 ID 기반 관리 |
| 단기 메모리 | ✅ 멀티턴 대화 | ✅ 세션 내 컨텍스트 | ✅ 세션 상태 |
| 장기 메모리 | ✅ 세션 간 영속, 에이전트 간 공유 | ✅ 3단계(추출/통합/검색) | ✅ MemoryBankService |
| 체크포인팅 | ✅ 비동기 태스크 관리 | ✅ 체크포인팅 | 제한적 |

### 5.2 보안 및 거버넌스

| 기능 | Bedrock AgentCore | Azure AI Foundry | Agent Engine |
|------|-------------------|-----------------|-------------|
| 네트워크 격리 | VPC 배포 | VNet/Private Endpoints | VPC-SC |
| 암호화 | KMS (at-rest), TLS (in-transit) | CMK 지원 | CMEK |
| ID 관리 | Cognito, Okta, Entra ID, Auth0 | Entra ID (Azure AD) | IAM |
| 접근 제어 정책 | Cedar 정책 언어 | RBAC | IAM 역할 |
| 컴플라이언스 | SOC 2, HIPAA, GDPR, ISO 27001 | SOC 2, HIPAA, GDPR, ISO 27001, FedRAMP | SOC 2, HIPAA, GDPR, ISO 27001 |

### 5.3 도구 통합 및 확장

| 기능 | Bedrock AgentCore | Azure AI Foundry | Agent Engine |
|------|-------------------|-----------------|-------------|
| MCP 지원 | ✅ Gateway 서비스 | ✅ (Agent Framework) | ✅ (ADK) |
| A2A 지원 | ✅ (Strands) | ✅ (Agent Framework) | ✅ (ADK) |
| 코드 실행 | ✅ Code Interpreter | ✅ | 제한적 |
| 브라우저 자동화 | ✅ Browser 서비스 | ✅ (Playwright) | ❌ |
| 외부 서비스 통합 | Salesforce, Zoom, Jira, Slack | Bing, Azure AI Search, Logic Apps | Google Search, Cloud Functions |

### 5.4 관측성 및 평가

| 기능 | Bedrock AgentCore | Azure AI Foundry | Agent Engine |
|------|-------------------|-----------------|-------------|
| 트레이싱 | OpenTelemetry 호환 | Application Insights | Cloud Trace |
| 평가 서비스 | ✅ Evaluations (자동 품질 평가) | ✅ (groundedness, relevance, coherence) | 제한적 |
| 실험 관리 | CloudWatch | Azure Experiments | Vertex AI Experiments |

---

## 6. 오픈소스 대안: 런타임 레이어

프레임워크 레벨에서는 ADK(Apache 2.0), Strands(Apache 2.0), Semantic Kernel(MIT) 모두 오픈소스이므로 Lock-in이 없습니다. 진짜 문제는 **매니지드 런타임을 오픈소스로 어떻게 대체하느냐**입니다.

### 6.1 LangGraph (MIT) — 프레임워크 + 체크포인팅 + 메모리

LangGraph 프레임워크 자체가 MIT 라이선스로 체크포인팅과 메모리를 내장하고 있습니다.

```python
from langgraph.graph import StateGraph
from langgraph.checkpoint.postgres import PostgresSaver

checkpointer = PostgresSaver.from_conn_string("postgresql://...")

graph = StateGraph(State)
graph.add_node("agent", agent_node)
# ... 그래프 정의

app = graph.compile(checkpointer=checkpointer)
```

**제공 기능:**

- Durable execution (장애 자동 복구)
- PostgreSQL 기반 체크포인팅 (`langgraph-checkpoint-postgres`, MIT)
- 단기/장기 메모리
- Human-in-the-Loop
- 스트리밍

**직접 구축해야 하는 것:** API 서버(FastAPI), 세션 라우팅, 인증/인가, 스케일링, 모니터링

⚠️ **주의**: LangGraph Platform(구 LangSmith Deployments)은 Elastic License 2.0으로, OSI 승인 오픈소스가 아닙니다. 셀프호스팅에는 라이선스 키가 필요하고, Enterprise 계약이 요구됩니다.

### 6.2 Aegra (Apache 2.0) — LangGraph Platform 드롭인 대체

LangGraph SDK와 API를 그대로 사용하면서 자체 인프라에서 PostgreSQL 영속성과 함께 운영할 수 있는 Apache 2.0 프로젝트입니다.

**제공 기능:**

- LangGraph SDK 호환 (기존 코드 수정 없이 사용)
- Agent Protocol 스펙 구현
- 체크포인트 포함 내구성 있는 대화 저장
- JWT/OAuth/Firebase 인증
- Docker Compose 5분 배포
- OpenTelemetry 관측성

**성숙도**: v0.8.x 초기 단계로 대규모 프로덕션 검증 사례가 부족합니다. PoC/평가 용도로 적합합니다.

### 6.3 Dify (수정 Apache 2.0) — 노코드/로우코드 플랫폼

⚠️ "Apache 2.0"으로 소개되지만 추가 제한 조건이 있습니다:

- 멀티테넌트 서비스 운영 시 별도 상업 라이선스 필요
- 프론트엔드 로고/저작권 정보 제거 불가

**제공 기능:** 시각적 워크플로 빌더, 빌트인 RAG, 지식 베이스, 100+ LLM 지원, 셀프호스팅

고객사에 멀티테넌트 SaaS로 제공하지 않는다면 사용 가능하지만, 금융권 등 라이선스 심사가 엄격한 환경에서는 주의가 필요합니다.

### 6.4 Mem0 (Apache 2.0) — 메모리 전용 레이어

에이전트 프레임워크가 아닌 **메모리 계층 전용** 오픈소스입니다.

- Apache 2.0 라이선스
- 온프레미스/프라이빗 클라우드/K8s 배포
- ADK, LangGraph, Strands 등과 조합하여 장기 메모리 레이어로 활용
- 50,000+ 개발자 사용 중

### 6.5 오픈소스 조합 전략 비교

| 조합 | 세션관리 | 체크포인팅 | 메모리 | API 서빙 | 라이선스 | 성숙도 |
|------|---------|-----------|--------|---------|---------|--------|
| LangGraph + FastAPI + PostgreSQL | 직접 구현 | ✅ 빌트인 | ✅ 빌트인 | 직접 구현 | MIT | 높음 |
| Aegra | ✅ | ✅ | ✅ | ✅ | Apache 2.0 | 초기 |
| ADK + Cloud Run + Firestore | 직접 구현 | 직접 구현 | 직접 구현 | Cloud Run | Apache 2.0 | 중간 |
| Strands + Lambda/EKS | 직접 구현 | 직접 구현 | 직접 구현 | Lambda/EKS | Apache 2.0 | 중간 |
| LangGraph Platform (셀프호스팅) | ✅ | ✅ | ✅ | ✅ | Elastic 2.0 ⚠️ | 매우 높음 |
| Dify 셀프호스팅 | ✅ | ✅ | ✅ | ✅ | 수정 Apache 2.0 ⚠️ | 높음 |

---

## 7. Lock-in 리스크 종합 매트릭스

### 7.1 프레임워크 레벨

| 프레임워크 | 라이선스 | 멀티 모델 | 멀티 클라우드 배포 | Lock-in |
|-----------|---------|-----------|-----------------|---------|
| ADK | Apache 2.0 | ✅ (Gemini 최적화) | ✅ Docker/K8s | 낮음 |
| Strands Agents | Apache 2.0 | ✅ (Bedrock, Anthropic, Ollama 등) | ✅ Docker/K8s | 낮음 |
| Semantic Kernel / Agent Framework | MIT | ✅ (OpenAI, Claude, Bedrock, Ollama 등) | ✅ Docker/K8s | 낮음 |
| LangGraph | MIT | ✅ | ✅ Docker/K8s | 낮음 |
| CrewAI | MIT | ✅ | ✅ Docker/K8s | 낮음 |

**결론**: 프레임워크 레벨에서는 3대 CSP 모두 오픈소스이며, Lock-in 리스크가 낮습니다.

### 7.2 매니지드 런타임 레벨

| 런타임 | 프레임워크 제한 | 모델 제한 | 데이터 이식성 | 대체 난이도 | Lock-in |
|--------|---------------|----------|-------------|-----------|---------|
| Bedrock AgentCore | 없음 (프레임워크 무관) | 없음 (모델 무관) | 중간 (Memory API) | 높음 | 중간~높음 |
| Azure AI Foundry | 없음 (Agent Framework 중심이나 타 지원) | OpenAI 중심 | 낮음 (M365 통합) | 높음 | 높음 |
| Agent Engine | ADK 중심 (타 지원) | Gemini 중심 | 중간 | 중간 (Cloud Run 대안) | 중간~높음 |

### 7.3 Lock-in 유형별 분석

**1. 모델 Lock-in**

- **AWS**: 가장 적음. 7개 이상의 모델 공급자 지원
- **Azure**: 가장 높음. Azure OpenAI가 중심이며 Claude 미지원
- **GCP**: 중간. Gemini 최적화이나 Model Garden을 통해 Claude, Llama 등 접근 가능

**2. 인프라 Lock-in**

- **AWS**: AgentCore의 microVM 세션 격리 모델은 AWS 고유
- **Azure**: Entra ID, M365, Copilot Studio 통합은 Azure 고유
- **GCP**: VPC-SC, CMEK는 GCP 고유이나, Cloud Run 배포로 런타임 Lock-in 회피 가능

**3. 데이터/메모리 Lock-in**

- **3사 모두**: 매니지드 메모리 서비스의 데이터를 다른 플랫폼으로 이식하기 어려움
- **완화 전략**: PostgreSQL/Redis 같은 표준 저장소에 메모리를 직접 저장하면 이식성 확보

**4. 생태계 Lock-in**

- **AWS**: AWS Lambda, Step Functions, CloudWatch 등과 깊은 통합
- **Azure**: M365, Teams, Dynamics, Power Platform과의 통합이 강력하지만 탈출 비용도 높음
- **GCP**: BigQuery, Firestore, Cloud Functions과의 통합

---

## 8. 실무 의사결정 프레임워크

### 8.1 어떤 CSP 런타임을 선택할 것인가

```
기존 클라우드가 있는가?
├── AWS 사용 중 → Bedrock AgentCore (프레임워크 무관 강점)
├── Azure 사용 중 → AI Foundry Agent Service (M365 통합 강점)
├── GCP 사용 중 → Agent Engine 또는 ADK + Cloud Run
└── 멀티 클라우드 / 없음
    ├── 모델 선택 자유도 우선 → AWS (가장 넓은 모델 선택지)
    ├── 엔터프라이즈 통합 우선 → Azure (M365/Copilot 생태계)
    └── 비용 우선 → GCP (Gemini 기준 가장 저렴)
```

### 8.2 오픈소스만으로 구축하고 싶다면

```
라이선스 리스크 제로가 필요한가?
├── Yes → LangGraph(MIT) + FastAPI + PostgreSQL
│         또는 ADK(Apache 2.0) + Docker/K8s
├── 초기이지만 올인원이 필요 → Aegra(Apache 2.0) 평가
└── No (약간의 제한 허용)
    ├── 노코드 필요 → Dify (수정 Apache 2.0)
    └── 프로덕션 검증 최우선 → LangGraph Platform (Elastic 2.0)
```

### 8.3 하이브리드 전략

가장 현실적인 접근은 **프레임워크는 오픈소스, 런타임은 CSP**입니다.

| 전략 | 프레임워크 | 런타임 | Lock-in | 비고 |
|------|-----------|--------|---------|------|
| GCP 최적 | ADK (Apache 2.0) | Cloud Run (Agent Engine 미사용) | 낮음 | 세션/메모리 직접 구현 필요 |
| AWS 최적 | Strands (Apache 2.0) | AgentCore Runtime만 사용 | 중간 | 모듈 선택적 사용으로 종속 최소화 |
| Azure 최적 | Semantic Kernel (MIT) | Azure AI Foundry | 중간~높음 | M365 생태계 활용 시 가치 극대화 |
| 멀티 클라우드 | LangGraph (MIT) | Docker/K8s + PostgreSQL | 최소 | 운영 부담 높지만 이식성 최대 |

---

## 9. 금융·규제 산업을 위한 추가 고려사항

금융, 의료, 공공 등 규제 산업에서는 추가적인 Lock-in 고려가 필요합니다.

### 데이터 주권

- **AWS**: 리전 선택 가능, Data residency 보장
- **Azure**: 리전 선택 + Data residency + Sovereign Cloud
- **GCP**: 리전 선택 + VPC-SC + Data Residency 제어

### 라이선스 감사 대비

```
완전한 오픈소스만 사용하고 싶다면:
✅ ADK (Apache 2.0)
✅ Strands (Apache 2.0)
✅ Semantic Kernel (MIT)
✅ LangGraph (MIT)
✅ CrewAI (MIT)
✅ Mem0 (Apache 2.0)
✅ PostgreSQL (PostgreSQL License)
✅ Redis (BSD)
✅ FastAPI (MIT)

⚠️ 주의가 필요한 것:
⚠️ LangGraph Platform — Elastic License 2.0 (OSI 미승인)
⚠️ Dify — 수정 Apache 2.0 (멀티테넌트 제한)
⚠️ 각 CSP 매니지드 서비스 — 상용 서비스 약관
```

### 탈출 비용 추정

| 전환 시나리오 | 프레임워크 전환 비용 | 런타임 전환 비용 | 데이터 마이그레이션 |
|-------------|-------------------|----------------|-----------------|
| GCP → AWS | 중간 (ADK → Strands 코드 전환) | 높음 (Agent Engine → AgentCore) | 높음 |
| AWS → GCP | 중간 (Strands → ADK 코드 전환) | 높음 (AgentCore → Agent Engine) | 높음 |
| CSP → 셀프호스팅 | 낮음 (오픈소스 프레임워크 유지) | 매우 높음 (전체 인프라 구축) | 중간 |
| LangGraph(MIT) 기반 | 없음 | 낮음 (Docker/K8s 이식) | 낮음 (PostgreSQL) |

---

## 10. 정리

### 3대 CSP의 공통 전략

1. **프레임워크는 오픈소스로 공개**하여 개발자 생태계를 확보
2. **매니지드 런타임에서 차별화**하여 수익 창출
3. **타사 프레임워크도 지원**하여 런타임 Lock-in 유도

### 실무 권장

- **프레임워크 선택은 자유롭게**: 3사 모두 오픈소스이므로 기술적 적합성으로 선택
- **런타임은 현재 CSP에 맞춰**: 기존 클라우드 인프라와의 통합 비용이 전환 비용보다 낮음
- **탈출 전략은 미리 준비**: 표준 프로토콜(MCP, A2A, OpenAPI) 활용, 메모리는 PostgreSQL 등 이식 가능한 저장소 사용
- **완전한 오픈소스 구축은 가능하지만 대가가 있음**: 세션 관리, 보안, 스케일링, 모니터링을 직접 구현해야 하는 운영 부담

> Agent Engine이 제공하는 수준의 "완전 매니지드 + 완전 오픈소스" 조합은 아직 시장에 존재하지 않습니다. 어딘가에서는 직접 구현하거나, 라이선스 제약을 받아들이거나, 또는 클라우드 비용을 지불해야 합니다. 이것이 현재 에이전트 인프라 생태계의 현실입니다.

---

## 참고 링크

- [Amazon Bedrock AgentCore 공식 문서](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/what-is-bedrock-agentcore.html)
- [Azure AI Foundry Agent Service](https://learn.microsoft.com/en-us/azure/ai-foundry/agents/)
- [Vertex AI Agent Engine](https://cloud.google.com/agent-builder/agent-engine)
- [Google ADK 공식 문서](https://google.github.io/adk-docs/)
- [Strands Agents SDK](https://strandsagents.com/)
- [Microsoft Agent Framework](https://devblogs.microsoft.com/semantic-kernel/introducing-microsoft-agent-framework/)
- [LangGraph GitHub](https://github.com/langchain-ai/langgraph)
- [Aegra GitHub](https://github.com/aegra-ai/aegra)
- [Mem0 GitHub](https://github.com/mem0ai/mem0)
- [Dify GitHub](https://github.com/langgenius/dify)
