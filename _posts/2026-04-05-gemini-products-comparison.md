---
layout: post
title: "Google Gemini 제품군 비교: Web, Workspace, Enterprise, Vertex AI + Gemini API"
date: 2026-04-05
tags: [gemini, vertex-ai, google-workspace, gemini-enterprise, 비교, google-cloud]
---

"Gemini"라는 이름이 붙은 Google 제품이 여러 개 있어서 고객분들이 자주 혼동하십니다. **이름은 비슷하지만 대상, 요금 방식, 기능이 완전히 다릅니다.** 이 글에서 한 번에 정리합니다.

---

## 전체 지도: Gemini 4종 분류

| 제품 | 결제 주체 | 요금 방식 | 대표 쓰임새 |
|------|----------|----------|------------|
| **Gemini Web** (무료) | 개인 | 무료 | 가벼운 챗봇 사용 |
| **Google AI Pro/Ultra** (개인 유료) | 개인 | 월 정액 (~$20/월) | 리서치, 코딩, 파일 분석 |
| **Gemini for Workspace** | 회사 (Workspace) | 사용자당 월 과금 | Gmail/Docs/Meet 내 업무용 AI |
| **Gemini Enterprise** | 회사 (GCP) | 라이선스당 월 ~$30~ | 엔터프라이즈 에이전트/포털 |
| **Vertex AI + Gemini API** | 개발팀 (GCP) | 토큰/리소스 사용량 | 커스텀 앱/에이전트 백엔드 |

---

## 1. Gemini Web (개인용)

gemini.google.com 또는 모바일 앱으로 접근하는 **개인용 AI 챗봇**입니다.

### 무료 vs Pro vs Ultra

| 구분 | 무료 | Google AI Pro | Ultra |
|------|------|--------------|-------|
| **가격** | 0원 | ~$20/월 | ~$250+/월 |
| **모델** | Gemini 2.5 Flash | Gemini 2.5 Pro | Pro + Deep Think/Ultra |
| **컨텍스트** | 짧은 대화 위주 | ~100만 토큰 (약 1,500페이지) | 더 긴 컨텍스트 |
| **파일 분석** | 제한적 | 대용량 문서/코드/PDF/동영상 | 고난도 분석 |
| **Deep Research** | X | O | O (강화) |
| **이미지/영상 생성** | 제한적 | 일부 | Veo 상위 기능 |
| **Gmail/Drive 연동** | X | O (개인 데이터) | O |

### 핵심 포인트

- **무료**는 Flash 모델 기반으로 간단한 질문/번역/요약 용도
- **Pro**는 장문 리포트, 대형 문서 분석, 고급 리서치에 적합
- **Ultra**는 복잡한 추론, 고난도 코딩, 전문 리서치용

> 개인용은 **"월 ~$20 내고 강력한 Gemini를 쓸 것인가, 무료로 가볍게 쓸 것인가"** 선택의 문제입니다.

---

## 2. Gemini for Google Workspace

Gmail, Docs, Sheets, Slides, Meet, Chat 등 **Workspace 앱 안에 붙는 AI 기능**입니다.

### 어디에서 뭘 해주나?

| Workspace 앱 | AI 기능 |
|-------------|---------|
| **Gmail** | 이메일 초안 작성, 요약, 답장 제안 |
| **Docs** | 문서 작성/수정/요약 보조 |
| **Sheets** | 수식 생성, 데이터 분류, 템플릿 |
| **Slides** | 슬라이드 자동 생성, 이미지 생성 |
| **Meet** | 회의 요약, 자동 노트, 번역 자막 |
| **Chat** | 대화 요약 |
| **Drive** | 자연어로 파일 검색 및 요약 |

### 요금 구조

**Google Workspace 요금제 + Gemini 애드온** 구조입니다.

| 결제 방식 | 설명 |
|----------|------|
| **탄력 요금제 (Flex)** | 월 단위, 사용자 수 가변, 일할 계산 |
| **연간/약정 요금제** | 1년 약정, Flex보다 사용자당 월 요금이 낮음 |

- Workspace Business Starter/Standard/Plus 등에 "Gemini 포함 플랜" 또는 "AI 애드온"으로 추가
- 결제 단위: **사용자 수 x 월 요금**, 중도 추가/삭제 시 일할 계산

### 개인용 Gemini Web과의 차이

같은 Gemini라도 Workspace 계정이면:

- **조직 데이터**(공유 드라이브, 조직 캘린더 등)에 접근 가능
- **관리 콘솔**에서 사용 제한, 로그/감사, DLP, Vault 등과 연계
- **기업 약관** + **데이터 미학습 보장** 적용

> Workspace용 Gemini는 **"직원당 월 X달러를 더 내고, 구글 업무 앱마다 AI를 붙인다"**라고 이해하면 됩니다.

---

## 3. Gemini Enterprise (GCP 기반)

Workspace에 붙는 도구가 아니라, **GCP에서 제공되는 엔터프라이즈 AI 포털/에이전트 플랫폼**입니다.

### Workspace용 Gemini와 뭐가 다른가?

| 구분 | Workspace 속 Gemini | Gemini Enterprise |
|------|-------------------|-------------------|
| **위치** | Gmail/Docs/Meet 안 | 별도 AI 포털 + GCP 콘솔 |
| **역할** | 개인의 생산성 도구 | 조직 차원의 에이전트 플랫폼 |
| **기능** | 앱 내 글쓰기/요약/분석 보조 | AI 에이전트 호스팅, 조직 데이터 인덱싱/검색 |
| **관리** | Workspace 관리 콘솔 | GCP 콘솔 (로깅, 모니터링, 정책) |
| **보안** | Workspace 수준 | VPC-SC, CMEK, FedRAMP High, HIPAA 등 |

### 요금

- **사용자당 월 ~$30부터** (Standard/Plus 등 SKU별 상이)
- 라이선스당 인덱스 스토리지(예: 75GiB) 포함
- 볼륨 디스카운트 가능

### 주요 기능

- 직원이 접속하는 **AI 허브/포털**
- 조직 데이터 **인덱싱 및 검색** (Drive, Calendar, Gmail, Chat 연동)
- 복잡한 업무용 **AI 에이전트 호스팅**
- Vertex AI MCP, Search, API 연동으로 워크플로 에이전트 정의
- 고급 보안: VPC-SC, CMEK, 액세스 투명성, 데이터 상주성

> Gemini Enterprise는 **"직원당 월 $30 이상 내고, 회사 전용 AI 포털과 에이전트 플랫폼을 산다"**는 개념입니다.

---

## 4. Vertex AI + Gemini API (커스텀 에이전트)

**Vertex AI에서 Gemini 모델을 API로 호출해 커스텀 에이전트/서비스를 직접 만드는** 개발자용 시나리오입니다.

### Enterprise와 뭐가 다른가?

| 구분 | Gemini Enterprise | Vertex AI + Gemini API |
|------|-------------------|----------------------|
| **성격** | 포털 + 플랫폼 라이선스 | 빌딩 블록 (API) |
| **요금** | 사용자당 월 과금 | 토큰/리소스 사용량 과금 |
| **UI** | 제공됨 (AI 포털) | 없음 (직접 만들어야 함) |
| **자유도** | 플랫폼 범위 내 | 완전 자유 |
| **적합 대상** | IT 관리자, 비즈니스 팀 | 개발자, 플랫폼 엔지니어 |

### 요금 (토큰 기반, 참고용)

| 모델 | 입력 (100만 토큰당) | 출력 (100만 토큰당) |
|------|-------------------|-------------------|
| **Gemini 2.5 Pro** | ~$1.25 - $2.50 | ~$10.00 |
| **Gemini 2.0 Flash** | ~$0.10 | ~$0.40 |
| **Gemini 2.0 Flash-Lite** | ~$0.025 | ~$0.10 |

> 가격은 수시로 변동됩니다. 최신 가격은 [Vertex AI Pricing](https://cloud.google.com/vertex-ai/generative-ai/pricing)에서 확인하세요.

### 주요 특징

- **구독이 아닌 사용량 과금**: 엔드유저 수가 아니라 API 호출량이 비용의 관건
- **완전 커스텀**: 프롬프트, 툴, 함수 호출, RAG, 외부 API/DB 연동을 마음대로 설계
- **B2C/B2B 서비스에 적합**: 고객 대상 앱, 사내 자동화 시스템 등에 Gemini를 임베드
- **멀티모달**: 텍스트, 이미지, 오디오, 비디오 입력 지원
- **롱 컨텍스트**: 최대 100만~200만 토큰
- **보안**: VPC-SC, CMEK, HIPAA, FedRAMP High 등 지원

> Enterprise는 **포털+플랫폼 라이선스**, Vertex AI는 **빌딩 블록(API) 과금**이라고 보시면 됩니다.

---

## 고객이 자주 혼동하는 질문 (FAQ)

### "Google AI Pro를 샀는데, 우리 회사 Gmail에도 자동으로 붙나요?"

**아닙니다.** Google AI Pro는 개인 계정용입니다. 회사 Workspace 메일에 AI를 붙이려면 **Gemini for Workspace** 라이선스를 별도로 구매해야 합니다.

### "Gemini Enterprise를 사면, Vertex AI API는 무료인가요?"

**별개입니다.** Enterprise는 직원용 포털/에이전트 라이선스이고, Vertex AI는 API 사용량 기반 과금입니다. 둘 다 비용이 발생합니다.

### "직원 50명 정도인데, Gmail/Docs에서 AI만 있으면 됩니다. 뭘 사야 하나요?"

**Google Workspace 플랜 + Gemini 포함 옵션**이 기본 선택입니다. 별도 AI 포털이 필요하면 그때 Gemini Enterprise를 검토하시면 됩니다.

### "커스텀 AI 챗봇을 우리 서비스에 넣고 싶은데요?"

**Vertex AI + Gemini API**가 적합합니다. 사용량 기반 과금이라 서비스 규모에 맞게 비용이 조절됩니다.

### "개인 Gemini Web과 Workspace용 Gemini, 기능이 같은가요?"

**UI는 비슷하지만 보안/감사/데이터 정책이 다릅니다.** Workspace 계정이면 조직 데이터 접근, 관리자 정책, 데이터 미학습 보장 등 기업용 약관이 적용됩니다.

### "Gemini for Workspace를 구독하면 개인용 Google AI Pro/Ultra를 회사 메일로 쓰는 것과 같나요?"

**완전히 같지는 않습니다.** Gemini for Workspace를 구독하면 Workspace 앱 내 AI 기능과 Gemini 앱(gemini.google.com) 접근 권한을 얻지만, 개인용 Pro/Ultra와 차이가 있습니다:

| 구분 | Google AI Pro (개인) | Gemini for Workspace |
|------|---------------------|---------------------|
| Gemini 웹 채팅 | O (Pro 모델) | O (Workspace 정책 적용) |
| Deep Research | O | 플랜에 따라 다름 |
| 모델 등급 | Pro/Ultra 선택 가능 | Google이 자동 선택 |
| 데이터 정책 | 소비자 약관 | 기업 약관 (데이터 미학습) |
| 조직 데이터 연동 | X (개인만) | O (공유 드라이브, 캘린더 등) |

**"Gemini 웹 채팅을 회사 계정으로 쓸 수 있다"는 맞지만, 모델 등급/기능 범위가 개인 Pro/Ultra와 동일하다고 보장되지는 않습니다.** 특히 Ultra급 기능(Deep Think, Veo 등)은 Workspace 플랜에 포함되지 않을 수 있습니다.

### "회사 메일(Workspace 계정)로 Google AI Pro/Ultra를 쓰려면 어떻게 하나요?"

개인이 Google One에서 결제하는 것과는 다릅니다. **Workspace 관리자가 Admin Console에서 라이선스를 할당하는 방식**입니다:

1. Admin Console > 구독 > Gemini 관련 라이선스 추가 구매
2. 사용자별로 라이선스 할당

**주의할 점:**
- 개인용 "Google AI Pro"와 Workspace용 "Google AI Pro"는 **SKU가 다릅니다**
- 관리자가 할당해도 **기업 데이터 정책**(데이터 미학습, 감사 로그 등)은 Workspace 기준으로 적용
- Ultra급이 Workspace SKU로 제공되는지는 아직 명확하지 않음 (Google이 수시로 업데이트 중)
- **정확한 SKU 이름과 포함 기능은 Google이 자주 변경하므로, 도입 시점에 Google 또는 메가존소프트를 통해 최신 SKU를 확인하는 것을 권장합니다.**

---

## 선택 가이드: 어떤 상황에 어떤 제품?

| 상황 | 추천 제품 |
|------|----------|
| 개인적으로 가볍게 AI 챗봇 사용 | Gemini Web (무료) |
| 개인적으로 심층 리서치/코딩에 활용 | Google AI Pro ($20/월) |
| 팀의 Gmail/Docs/Meet 생산성 향상 | Gemini for Workspace |
| 전사 AI 포털 + 에이전트 플랫폼 도입 | Gemini Enterprise |
| 커스텀 AI 앱/에이전트를 직접 개발 | Vertex AI + Gemini API |

**핵심은 이것입니다:**
- "조직에서 직원들이 쓸 포털형 AI"를 원하면 → **Enterprise**
- "고객/직원 대상 커스텀 앱/에이전트를 직접 개발"하고 싶으면 → **Vertex AI API**
- "Workspace 앱에서 바로 AI 보조"가 필요하면 → **Gemini for Workspace**

도입에 대한 자세한 문의는 [메가존소프트](https://www.megazonesoft.com)를 통해 연락해 주세요.
