---
layout: post
title: "Google Gemini 제품 라인업 완전 정리: 일반 Gemini, Workspace, Enterprise, Vertex AI"
date: 2026-02-14
tags: [gemini, vertex ai, gemini workspace, gemini enterprise, google cloud]
---

Google의 AI 브랜드 **Gemini**는 이제 단일 제품이 아니라 하나의 거대한 제품 패밀리입니다. 소비자용 챗봇부터 엔터프라이즈 에이전트 플랫폼, 개발자용 API까지 — 이름은 비슷하지만 대상, 기능, 과금 체계가 모두 다릅니다.

이 글에서는 **일반 Gemini(소비자용)**, **Gemini for Google Workspace**, **Gemini Enterprise(구 Agentspace)**, **Vertex AI의 Gemini API**를 비교하고 각각의 용도를 정리합니다.

---

## 1. Google Gemini (일반 소비자용)

[gemini.google.com](https://gemini.google.com)에서 누구나 사용할 수 있는 AI 어시스턴트입니다. ChatGPT, Claude와 같은 포지션의 대화형 AI 서비스입니다.

### 주요 특징

- **무료 플랜**: Gemini Flash 모델 기반, 32K 토큰 컨텍스트 윈도우
- **유료 플랜 (Google AI Pro/Ultra)**: Gemini 3 Pro 모델, 100만 토큰 컨텍스트 윈도우
- 텍스트 생성, 이미지 생성, 웹 검색, 요약
- Gemini Live (음성 대화)
- 코드 실행 및 편집

### 대상 사용자

일반 소비자, 학생, 개인 사용자. 일상적인 질문 답변, 글쓰기 보조, 간단한 코딩 도움이 필요한 경우.

### 과금

| 플랜 | 가격 | 모델 |
|------|------|------|
| 무료 | $0 | Gemini Flash |
| AI Plus | ~$7.99/월 | Gemini 3 Pro |
| AI Ultra | ~$24.99/월 | Gemini 3 Ultra |

---

## 2. Gemini for Google Workspace

Google Workspace(Gmail, Docs, Sheets, Slides, Meet 등)에 **내장된** AI 기능입니다. 별도의 앱이 아니라 기존 Workspace 앱 안에서 동작합니다.

### 주요 기능

- **Gmail**: 이메일 초안 작성, 요약, 톤 조정
- **Docs**: 문서 작성 보조, 요약, 리라이팅
- **Sheets**: 수식 생성, 데이터 정리, 피벗 테이블 자동 생성
- **Slides**: 프레젠테이션 이미지 생성, 슬라이드 요약
- **Meet**: 실시간 자막, 미팅 노트 자동 생성, 배경 생성

### 대상 사용자

Workspace를 사용하는 비즈니스 사용자. 매일 이메일, 문서, 스프레드시트를 다루는 업무 환경에서 생산성을 높이고 싶은 경우.

### 핵심 포인트

- **개별 앱 내에서** 동작 (앱 간 크로스 연동은 제한적)
- Google Workspace 라이선스에 Gemini add-on으로 추가
- 엔터프라이즈 보안/컴플라이언스 정책 적용

---

## 3. Gemini Enterprise (구 Agentspace)

2024년 말 **Agentspace**로 출시되었다가 **Gemini Enterprise**로 리브랜딩된 엔터프라이즈 에이전트 플랫폼입니다. 단순한 AI 어시스턴트가 아니라 **자율적으로 다단계 작업을 수행하는 AI 에이전트**를 구축하고 관리하는 플랫폼입니다.

### Workspace와의 결정적 차이

| | Gemini for Workspace | Gemini Enterprise |
|---|---|---|
| **동작 범위** | 개별 Workspace 앱 내 | 조직 전체 시스템 횡단 |
| **연동** | Workspace 앱 한정 | Google Workspace + Microsoft 365 + Jira, Confluence, ServiceNow 등 |
| **에이전트** | 없음 (단순 보조) | 자율 에이전트 생성/실행 |
| **검색** | 앱 내 검색 | 엔터프라이즈 통합 검색 (Chrome 주소창에서도 가능) |

### 사전 구축된(Pre-built) 에이전트

- **Deep Research**: 웹 및 사내 데이터를 종합적으로 조사하여 리포트 생성
- **Data Insights**: SQL 없이 BigQuery 데이터에서 인사이트 추출
- **Idea Generation**: 토너먼트 방식 프레임워크로 비즈니스 아이디어 생성/순위 매기기
- **NotebookLM Enterprise**: 복잡한 정보 요약 및 연구 도구
- **Gemini Code Assist**: 개발자용 코딩 보조

### 커스텀 에이전트 구축

- **No-code Agent Designer**: 코딩 없이 에이전트 생성
- **Agent Development Kit (ADK)**: Vertex AI와 연계한 풀코드 에이전트 개발
- **A2A (Agent-to-Agent) 프로토콜**: 서드파티 에이전트와의 상호운용

### 에디션

| 에디션 | 대상 |
|--------|------|
| Business | 중소기업, 소규모 팀 |
| Standard | 대기업 일반 직원 |
| Plus | 고급 AI 도구 및 에이전트 빌딩이 필요한 팀 |
| Frontline | 현장 근무자 (비용 효율적) |

### 대상 사용자

IT 관리자, 비즈니스 프로세스 오너, CIO/CTO. 조직 차원에서 AI 에이전트를 도입하여 복잡한 워크플로우를 자동화하고 싶은 경우.

---

## 4. Vertex AI의 Gemini API와 부속 컴포넌트

**개발자와 데이터 사이언티스트**를 위한 풀 매니지드 AI 개발 플랫폼입니다. Gemini 모델을 포함한 200개 이상의 파운데이션 모델에 프로그래매틱하게 접근할 수 있습니다.

### 핵심 컴포넌트

#### Vertex AI Studio

모델을 **대화형 UI**에서 테스트하고 프롬프트를 실험할 수 있는 인터페이스입니다. 코드를 작성하기 전에 프롬프트 엔지니어링을 반복 실험하기에 적합합니다.

#### Gemini API (generateContent / streamGenerateContent)

Gemini 모델을 애플리케이션에 통합하기 위한 REST/gRPC API입니다.

- **멀티모달 입력**: 텍스트, 이미지, 비디오, 오디오를 동시에 처리
- **파라미터 제어**: temperature, top-P, top-K, safety settings, response schema
- **시스템 인스트럭션**: 모델의 페르소나/행동 규칙 설정
- **Function Calling**: 외부 도구/API와 연동하여 모델이 액션 수행
- **캐시드 콘텐츠**: 반복 요청 시 비용 및 레이턴시 절감
- **스트리밍 지원**: 실시간 응답 생성

#### Model Garden

200개 이상의 모델을 탐색하고 배포할 수 있는 카탈로그입니다.

- **퍼스트파티 모델**: Gemini 시리즈, Imagen(이미지 생성), Veo(비디오 생성), Chirp(음성 인식)
- **오픈 모델**: Gemma, Llama, Mistral 등
- **서드파티 모델**: Anthropic Claude 패밀리
- **원클릭 배포**: GPU 기반 엔드포인트에 바로 배포 가능

#### Agent Builder

AI 에이전트를 구축하기 위한 프레임워크입니다. 검색 기반 RAG, 대화형 에이전트, 멀티 에이전트 오케스트레이션 등을 지원합니다.

#### 기타 주요 기능

- **튜닝(Fine-tuning)**: 커스텀 데이터로 모델 미세 조정
- **평가(Evaluation)**: 모델 성능 평가 자동화
- **프로비저닝된 처리량(Provisioned Throughput)**: 안정적인 추론 성능 보장
- **VPC-SC / CMEK**: 엔터프라이즈급 보안 및 데이터 주권

### 대상 사용자

ML 엔지니어, 백엔드 개발자, 데이터 사이언티스트. AI 모델을 **자사 제품/서비스에 직접 통합**하거나, 커스텀 모델을 훈련/배포해야 하는 경우.

### 과금

사용량 기반 과금(pay-as-you-go). 입력/출력 토큰 수, 이미지/비디오 처리량, 엔드포인트 컴퓨팅 리소스 기준으로 과금됩니다.

---

## 한눈에 보는 비교표

| 구분 | 일반 Gemini | Gemini for Workspace | Gemini Enterprise | Vertex AI Gemini API |
|------|------------|---------------------|-------------------|---------------------|
| **대상** | 일반 소비자 | 비즈니스 사용자 | 조직/IT 관리자 | 개발자/ML 엔지니어 |
| **인터페이스** | 웹/앱 채팅 | Workspace 앱 내장 | 전용 플랫폼 + Chrome | API / SDK / Studio |
| **핵심 가치** | 개인 생산성 | 업무 생산성 | 엔터프라이즈 자동화 | AI 제품 개발 |
| **에이전트** | X | X | O (핵심 기능) | O (Agent Builder) |
| **커스터마이징** | X | X | No-code + Full-code | Fine-tuning + Full-code |
| **데이터 연동** | 웹 검색 | Workspace 앱 | 사내 시스템 전체 | 자유 (API 기반) |
| **과금** | 무료/구독 | Workspace add-on | 에디션별 구독 | 사용량 기반 |

---

## 데이터 보안 및 학습 데이터 재사용 비교

엔터프라이즈 환경에서 AI를 도입할 때 가장 중요한 질문 중 하나는 **"내 대화 내역과 데이터가 LLM 모델 학습에 재사용되는가?"** 입니다. 제품별로 정책이 크게 다르므로 반드시 확인해야 합니다.

### 제품별 데이터 학습 재사용 정책

| 구분 | 일반 Gemini | Gemini for Workspace | Gemini Enterprise | Vertex AI Gemini API |
|------|------------|---------------------|-------------------|---------------------|
| **모델 학습에 데이터 사용** | 조건부 사용 | **사용 안 함** | **사용 안 함** | **사용 안 함** |
| **사람에 의한 리뷰** | 가능 (활동 기록 ON 시) | 안 함 | 안 함 | 안 함 (남용 모니터링 제외) |
| **데이터 보존 기간** | 활동 기록 설정에 따라 다름 | Workspace 정책 적용 | 서비스 약관 적용 | 제로 데이터 보존 (Zero Retention) 가능 |
| **옵트아웃 가능** | O (활동 기록 OFF) | 해당 없음 (기본 보호) | 해당 없음 (기본 보호) | 해당 없음 (기본 보호) |

### 일반 Gemini (소비자용)

- **"활동 기록(Keep Activity)"이 켜져 있으면** 대화 내역이 저장되고, Google이 서비스 개선을 위해 사람이 리뷰할 수 있습니다.
- 활동 기록을 끄면 대화가 제한된 기간 동안만 임시 보존된 후 삭제됩니다.
- **기업 데이터를 일반 Gemini에 입력하는 것은 보안상 권장하지 않습니다.**

### Gemini for Workspace / Gemini Enterprise

- **유료 서비스에서 고객 데이터는 Google의 파운데이션 모델 학습에 사용되지 않습니다.**
- 사람에 의한 리뷰도 수행되지 않으며, 데이터는 조직 외부로 공유되지 않습니다.
- Google Workspace 서비스 약관 및 **데이터 처리 부속서(DPA, Data Processing Addendum)** 가 적용되어 엔터프라이즈급 데이터 보호를 보장합니다.
- 단, **Starter Edition(무료 티어)** 은 예외로, 서비스 개선 목적으로 데이터가 활용될 수 있습니다(옵트아웃 가능).

### Vertex AI Gemini API

- **가장 강력한 데이터 보호 정책**을 제공합니다.
- 프롬프트와 응답은 모델 학습/미세 조정에 **절대 사용되지 않습니다.**
- **제로 데이터 보존(Zero Data Retention)** 을 지원하여 고객 데이터를 서버에 남기지 않는 설정이 가능합니다.
- 남용 모니터링을 위해 최대 55일간 프롬프트가 보존될 수 있으나, 이마저도 예외 신청을 통해 비활성화할 수 있습니다.
- Google Cloud의 **VPC 서비스 제어(VPC-SC)**, **고객 관리 암호화 키(CMEK)** 와 결합하여 최고 수준의 데이터 주권을 확보할 수 있습니다.

### 핵심 요약

> **엔터프라이즈 환경에서는 일반 Gemini(소비자용)를 업무에 사용하지 말아야 합니다.** Gemini for Workspace, Gemini Enterprise, Vertex AI는 모두 고객 데이터를 모델 학습에 사용하지 않으며, 특히 Vertex AI는 제로 데이터 보존까지 지원합니다. 데이터 보안이 최우선인 조직이라면 **Vertex AI** 또는 **Gemini Enterprise**를 선택하세요.

---

## 어떤 걸 선택해야 할까?

### "우리 직원들이 AI로 이메일, 문서 작업을 더 빨리 하게 하고 싶다"
→ **Gemini for Google Workspace**

### "사내 시스템을 횡단하는 자동화된 AI 에이전트를 도입하고 싶다"
→ **Gemini Enterprise**

### "AI 모델을 우리 제품/서비스에 직접 넣고 싶다"
→ **Vertex AI Gemini API**

### "개인적으로 AI 어시스턴트를 사용하고 싶다"
→ **일반 Gemini (gemini.google.com)**

---

## 함께 쓰면 더 강력하다

이 제품들은 서로 배타적이지 않습니다. 오히려 함께 사용할 때 시너지가 납니다.

- **Vertex AI**에서 커스텀 모델을 만들고 → **Gemini Enterprise**에 에이전트로 배포
- **Gemini Enterprise**의 에이전트가 **Workspace**의 데이터를 검색하고 활용
- **Vertex AI Agent Builder**로 만든 에이전트를 **A2A 프로토콜**로 Gemini Enterprise에 연결

Google이 지향하는 것은 결국 **하나의 AI 생태계** 안에서 소비자, 비즈니스 사용자, 개발자 모두를 아우르는 통합 경험입니다. 각 제품의 역할과 경계를 이해하면 조직에 가장 적합한 AI 전략을 설계하는 데 큰 도움이 될 것입니다.
