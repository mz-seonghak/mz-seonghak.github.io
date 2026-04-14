---
title: "Google Gemini 제품군 비교"
date: 2026-04-05
tags: [gemini, vertex-ai, gemini-workspace, gemini-enterprise, google-cloud, 비교]
description: "Web, Workspace, Enterprise, Vertex AI + Gemini API — 이름은 같지만 전혀 다른 Gemini 제품 4종을 한눈에 비교합니다."
theme: white
---

# Google Gemini 제품군 비교

Web · Workspace · Enterprise · Vertex AI API

---

## 전체 지도: Gemini 제품 분류

| 제품 | 결제 주체 | 요금 방식 | 대표 쓰임새 |
|------|----------|----------|------------|
| **Gemini Web** (무료) | 개인 | 무료 | 가벼운 챗봇 |
| **Google AI Pro/Ultra** | 개인 | 월 정액 (~$20) | 리서치, 코딩, 파일 분석 |
| **Gemini for Workspace** | 회사 (GWS) | 사용자당 월 | Gmail/Docs/Meet 업무 AI |
| **Gemini Enterprise** | 회사 (GCP) | 라이선스당 월 ~$30~ | 에이전트/포털 |
| **Vertex AI + Gemini API** | 개발팀 (GCP) | 토큰 사용량 | 커스텀 앱/에이전트 |

> 같은 "Gemini"지만 **대상/요금/기능/데이터 정책**이 모두 다릅니다

---

## 1. Gemini Web (개인용)

| 구분 | 무료 | Google AI Pro | Ultra |
|------|------|--------------|-------|
| **가격** | 0원 | ~$20/월 | ~$250+/월 |
| **모델** | 2.5 Flash | 2.5 Pro | Pro + Deep Think |
| **컨텍스트** | 짧은 대화 | ~100만 토큰 | 더 긴 컨텍스트 |
| **파일 분석** | 제한적 | 대용량 문서/코드/PDF | 고난도 분석 |
| **Deep Research** | X | O | O (강화) |
| **Gmail/Drive** | X | O (개인) | O |

> **"월 ~$20 내고 강력한 Gemini를 쓸 것인가"** 선택의 문제

---

## 2. Gemini for Workspace

**Workspace 앱 안에 붙는 AI 기능**

- **Gmail** — 이메일 초안, 요약, 답장 제안
- **Docs** — 문서 작성/수정/요약
- **Sheets** — 수식 생성, 데이터 분류
- **Meet** — 회의 요약, 번역 자막
- **Drive** — 자연어 파일 검색/요약

요금: **Workspace 플랜 + Gemini 애드온** (사용자당 월, Flex/연간)

> **"직원당 월 X달러, 구글 업무 앱마다 AI를 붙인다"**

---

## 3. Gemini Enterprise (GCP)

| 구분 | Workspace 속 Gemini | Gemini Enterprise |
|------|-------------------|-------------------|
| **위치** | Gmail/Docs/Meet 안 | 별도 AI 포털 + GCP 콘솔 |
| **역할** | 개인 생산성 도구 | 조직 에이전트 플랫폼 |
| **기능** | 앱 내 글쓰기/요약 | 에이전트 호스팅, 데이터 인덱싱 |
| **보안** | Workspace 수준 | VPC-SC, CMEK, FedRAMP, HIPAA |

요금: **사용자당 월 ~$30부터** (SKU별 상이)

> **"회사 전용 AI 포털과 에이전트 플랫폼을 산다"**

---

## 4. Vertex AI + Gemini API

| 구분 | Enterprise | Vertex AI API |
|------|-----------|--------------|
| **성격** | 포털+플랫폼 | 빌딩 블록(API) |
| **요금** | 사용자당 월 | 토큰 사용량 |
| **UI** | 제공됨 | 직접 구축 |
| **자유도** | 플랫폼 범위 | 완전 자유 |

| 모델 | 입력/1M | 출력/1M |
|------|--------|--------|
| 2.5 Pro | ~$1.25 | ~$10 |
| 2.0 Flash | ~$0.10 | ~$0.40 |
| Flash-Lite | ~$0.025 | ~$0.10 |

> Enterprise = **포털+라이선스**, Vertex AI = **빌딩 블록(API) 과금**

---

## Workspace Gemini vs 개인 Pro/Ultra

| 구분 | Google AI Pro (개인) | Gemini for Workspace |
|------|---------------------|---------------------|
| Gemini 웹 채팅 | O (Pro 모델) | O (Workspace 정책) |
| Deep Research | O | 플랜에 따라 다름 |
| 모델 등급 | Pro/Ultra 선택 | Google 자동 선택 |
| 데이터 정책 | 소비자 약관 | 기업 약관 (미학습) |
| 조직 데이터 | X | O |

회사 계정으로 Pro급을 쓰려면 → **관리자가 Admin Console에서 별도 SKU 할당**

---

## FAQ: 고객이 자주 혼동하는 질문

**Q. Google AI Pro를 샀는데, 회사 Gmail에도 붙나요?**
→ 아닙니다. 별도 Workspace 라이선스 필요

**Q. Enterprise를 사면 Vertex AI API도 무료?**
→ 별개입니다. 둘 다 비용 발생

**Q. 직원 50명, Gmail AI만 필요합니다.**
→ Workspace + Gemini 포함 옵션

**Q. Workspace Gemini = 개인 Pro/Ultra?**
→ 완전히 같지 않음. 모델 등급/기능 범위 차이

---

## 선택 가이드

| 상황 | 추천 |
|------|------|
| 개인 가벼운 사용 | Gemini Web (무료) |
| 개인 심층 리서치/코딩 | Google AI Pro |
| 팀 Gmail/Docs 생산성 | Gemini for Workspace |
| 전사 AI 포털/에이전트 | Gemini Enterprise |
| 커스텀 앱/에이전트 개발 | Vertex AI + Gemini API |

---

# 감사합니다

문의: [megazonesoft.com](https://www.megazonesoft.com)
