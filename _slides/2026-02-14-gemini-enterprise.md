---
title: "Google Gemini 제품 라인업 완전 정리"
date: 2026-02-14
tags: [gemini, vertex ai, gemini workspace, gemini enterprise, google cloud]
description: "일반 Gemini, Workspace, Enterprise, Vertex AI — 이름은 비슷하지만 대상, 기능, 과금이 모두 다른 Gemini 패밀리를 한눈에 비교합니다."
theme: white
---

# Google Gemini 제품 라인업 완전 정리

일반 Gemini · Workspace · Enterprise · Vertex AI

---

## Gemini는 하나가 아니다

Google의 AI 브랜드 **Gemini**는 이제 단일 제품이 아니라 **거대한 제품 패밀리**

- 소비자용 챗봇
- Workspace 내장 AI
- 엔터프라이즈 에이전트 플랫폼
- 개발자용 API

> 이름은 비슷하지만 **대상, 기능, 과금 체계**가 모두 다릅니다

---

## 1. 일반 Gemini (소비자용)

[gemini.google.com](https://gemini.google.com) — ChatGPT, Claude와 같은 포지션

### 주요 특징
- 텍스트 생성, 이미지 생성, 웹 검색, 요약
- Gemini Live (음성 대화)
- 코드 실행 및 편집

### 과금

| 플랜 | 가격 | 모델 |
|------|------|------|
| 무료 | $0 | Gemini Flash |
| AI Plus | ~$7.99/월 | Gemini 3 Pro |
| AI Ultra | ~$24.99/월 | Gemini 3 Ultra |

---

## 2. Gemini for Google Workspace

Workspace 앱(Gmail, Docs, Sheets, Slides, Meet)에 **내장된** AI

- **Gmail** — 이메일 초안 작성, 요약, 톤 조정
- **Docs** — 문서 작성 보조, 요약, 리라이팅
- **Sheets** — 수식 생성, 데이터 정리, 피벗 자동 생성
- **Slides** — 이미지 생성, 슬라이드 요약
- **Meet** — 실시간 자막, 미팅 노트 자동 생성

> 별도 앱이 아닌 **기존 Workspace 앱 안에서** 동작

---

## 3. Gemini Enterprise (구 Agentspace)

2024년 말 **Agentspace** → **Gemini Enterprise**로 리브랜딩

단순 AI 어시스턴트가 아닌 **자율적 다단계 작업 수행 AI 에이전트** 플랫폼

---

## Workspace vs Enterprise 차이

| | Gemini for Workspace | Gemini Enterprise |
|---|---|---|
| **동작 범위** | 개별 Workspace 앱 내 | 조직 전체 시스템 횡단 |
| **연동** | Workspace 앱 한정 | Workspace + M365 + Jira, Confluence 등 |
| **에이전트** | 없음 (단순 보조) | 자율 에이전트 생성/실행 |
| **검색** | 앱 내 검색 | 엔터프라이즈 통합 검색 |

---

## Enterprise: Pre-built 에이전트

- **Deep Research** — 웹 + 사내 데이터 종합 조사 → 리포트 생성
- **Data Insights** — SQL 없이 BigQuery에서 인사이트 추출
- **Idea Generation** — 토너먼트 방식 아이디어 생성/순위
- **NotebookLM Enterprise** — 복잡한 정보 요약 및 연구
- **Gemini Code Assist** — 개발자용 코딩 보조

---

## Enterprise: 커스텀 에이전트 구축

- **No-code Agent Designer** — 코딩 없이 에이전트 생성
- **Agent Development Kit (ADK)** — Vertex AI 연계 풀코드 개발
- **A2A 프로토콜** — 서드파티 에이전트와 상호운용

### 에디션

| 에디션 | 대상 |
|--------|------|
| Business | 중소기업, 소규모 팀 |
| Standard | 대기업 일반 직원 |
| Plus | 고급 AI 도구 + 에이전트 빌딩 |
| Frontline | 현장 근무자 (비용 효율적) |

---

## 4. Vertex AI의 Gemini API

**개발자와 데이터 사이언티스트**를 위한 풀 매니지드 AI 개발 플랫폼

200개 이상의 파운데이션 모델에 프로그래매틱 접근

---

## Vertex AI: 핵심 컴포넌트

- **Vertex AI Studio** — 대화형 UI로 프롬프트 실험
- **Gemini API** — 멀티모달, Function Calling, 스트리밍
- **Model Garden** — 200+ 모델 카탈로그 (Gemini, Gemma, Llama, Claude 등)
- **Agent Builder** — RAG, 대화형 에이전트, 멀티 에이전트 오케스트레이션

---

## Vertex AI: 엔터프라이즈 기능

- **튜닝(Fine-tuning)** — 커스텀 데이터로 모델 미세 조정
- **평가(Evaluation)** — 모델 성능 평가 자동화
- **프로비저닝된 처리량** — 안정적인 추론 성능 보장
- **VPC-SC / CMEK** — 엔터프라이즈급 보안 및 데이터 주권

> **과금**: 사용량 기반 (입력/출력 토큰 수, 컴퓨팅 리소스)

---

## 한눈에 보는 비교표

| 구분 | 일반 Gemini | Workspace | Enterprise | Vertex AI |
|------|-----------|-----------|------------|-----------|
| **대상** | 소비자 | 비즈니스 사용자 | 조직/IT관리자 | 개발자/ML |
| **인터페이스** | 웹/앱 채팅 | 앱 내장 | 전용 플랫폼 | API/SDK |
| **에이전트** | X | X | O | O |
| **커스터마이징** | X | X | No-code + Code | Fine-tuning |
| **과금** | 무료/구독 | Add-on | 에디션 구독 | 사용량 |

---

## 어떤 걸 선택해야 할까?

**"직원들이 이메일, 문서 작업을 더 빨리 하게 하고 싶다"**
→ Gemini for Google Workspace

**"사내 시스템 횡단 자동화 AI 에이전트를 도입하고 싶다"**
→ Gemini Enterprise

**"AI 모델을 우리 제품/서비스에 직접 넣고 싶다"**
→ Vertex AI Gemini API

**"개인적으로 AI를 사용하고 싶다"**
→ 일반 Gemini

---

## 함께 쓰면 더 강력하다

이 제품들은 **서로 배타적이지 않습니다**

- **Vertex AI**에서 커스텀 모델 → **Enterprise**에 에이전트로 배포
- **Enterprise** 에이전트가 **Workspace** 데이터를 검색·활용
- **Agent Builder**로 만든 에이전트를 **A2A**로 Enterprise에 연결

> Google이 지향하는 것은 **하나의 AI 생태계** 안에서 소비자, 비즈니스 사용자, 개발자 모두를 아우르는 통합 경험

---

# 감사합니다

질문이 있으시면 편하게 말씀해 주세요
