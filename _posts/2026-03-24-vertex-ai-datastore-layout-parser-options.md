---
layout: post
title: "Vertex AI Datastore Layout Parser 옵션 선택 가이드: ERP 매뉴얼 인덱싱 실전 경험"
date: 2026-03-24
tags: [vertex-ai, agent-builder, datastore, layout-parser, document-ai, gemini, rag, google-cloud]
---

Vertex AI Agent Builder의 Datastore에 문서를 인덱싱할 때, **어떤 Parser를 쓰고 어떤 옵션을 켤 것인가**는 RAG 시스템의 답변 품질을 좌우하는 핵심 결정입니다.

특히 ERP 매뉴얼처럼 다이어그램, 차트, 이미지가 많은 문서를 다룰 때는 Parser 선택이 더욱 중요해집니다. 이 글에서는 Layout Parser의 3가지 옵션에 대한 실전 테스트 경험과 Digital Parser와의 비교 결과를 정리합니다.

---

## 1. Parser 종류와 Layout Parser 옵션 이해

Vertex AI Datastore는 문서를 청크로 나누고 인덱싱할 때 Parser를 사용합니다. 대표적으로 **Digital Parser**와 **Layout Parser** 두 가지가 있습니다.

| Parser | 특징 | 적합한 문서 |
|--------|------|------------|
| Digital Parser | 기본 텍스트 추출, 저렴한 비용 | 텍스트 위주의 단순 문서 |
| Layout Parser | 문서 레이아웃(제목, 소제목, 단락) 구조 이해 | 복잡한 레이아웃의 문서 |

Layout Parser에는 다음 **3가지 추가 옵션**이 있습니다.

### 1-1. Table Annotation

테이블(표)의 구조를 인식하고 설명을 생성하여 청크에 할당합니다. 표가 포함된 문서에서 정확한 데이터 추출이 필요할 때 유용합니다.

### 1-2. Image Annotation

이미지에 대한 설명(description)을 생성하고 해당 이미지를 청크에 할당합니다. 다만 **텍스트가 메인인 이미지**(카드뉴스 형태 등)는 인식이 가능하지만, 순수한 그림이나 다이어그램의 유기적인 내용까지 이해하는 데는 한계가 있습니다.

### 1-3. Gemini Enhancement (Public Preview)

기존 Layout Parser 위에 **Gemini 모델이 추가 분석 레이어**로 동작합니다. 기존 Layout Parser가 가지고 있던 한계점(PPT 내 차트 이해 불가, 다이어그램 유기적 내용 이해 불가, 표 레이아웃 잘못 해석 등)을 Gemini의 멀티모달 능력으로 보완합니다.

> 참고: [Gemini 레이아웃 파서로 문서 처리 | Document AI | Google Cloud](https://docs.cloud.google.com/document-ai/docs/layout-parse-chunk?hl=ko#parser-types)

---

## 2. Layout Parser 옵션별 한계점 (Gemini Enhancement 미사용 시)

2025년 11월 기준 테스트에서 Gemini Enhancement 없이 Layout Parser만 사용했을 때 확인된 한계점입니다.

| 문서 유형 | 한계점 |
|-----------|--------|
| PPT | 텍스트는 이해 가능하나, **내부 차트는 이해 불가** |
| 다이어그램 | 내부 텍스트는 인식하나, **요소 간 관계나 흐름 이해 불가** |
| 그림/일러스트 | Image Annotation을 켜도 순수 이미지는 **인식 정확도 낮음** |
| 표(Table) | 대체로 인식하지만, **복잡한 병합 셀 등에서 레이아웃 오류 발생** |

즉 Table Annotation과 Image Annotation을 모두 켜더라도 정확도가 충분하지 않은 경우가 있었습니다. 다만 Digital Parser에 비하면 **확연히 나은 성능**을 보였습니다.

---

## 3. Digital Parser vs Layout Parser 정확도 비교

실제 어떤 기업의 운영 환경에서 동일한 문서 셋을 대상으로 Digital Parser와 Layout Parser의 답변 정확도를 비교한 결과입니다.

| 문서 유형 | Digital Parser | Layout Parser | 개선폭 |
|-----------|---------------|---------------|--------|
| 메일 | 50% | **85%** | +35%p |
| PPT | 10% 내외 | **55%** | +45%p |
| PDF | 30% | **86.7%** | +56.7%p |
| 품의 데이터 | 30% 내외 | **70%** | +40%p |

모든 문서 유형에서 Layout Parser가 **압도적인 정확도 향상**을 보여줍니다. 특히 PDF 문서에서는 30%에서 86.7%로 거의 3배 가까운 개선이 있었습니다.

---

## 4. 모든 옵션을 켜야 하는가?

### 테스트 결과: 모두 활성화가 최적

Table Annotation, Image Annotation, Gemini Enhancement를 **모두 활성화**했을 때 답변 품질이 가장 좋았습니다.

다만 비용 측면에서 고려할 점이 있습니다.

| 항목 | 내용 |
|------|------|
| 토큰 비용 증가 | 옵션 모두 활성화 시 평균 **약 17% 증가** |
| Parser 비용 | Layout Parser 자체가 Digital Parser보다 비용이 높음 |
| 재인덱싱 비용 | 설정 변경 시 새 Datastore를 생성해야 하므로 파싱/인덱싱 비용 재발생 |

### 권장 전략

실질적으로 상용 수준의 답변 품질을 달성하려면 **Layout Parser + 전체 옵션 활성화**가 사실상 필수입니다. 비용은 증가하지만 답변 품질 차이가 워낙 크기 때문에, 정확한 정보 제공이 중요한 비즈니스 환경에서는 비용 대비 효과가 충분합니다.

```
권장 설정:
├── Parser: Layout Parser (필수)
├── Table Annotation: ON (표가 포함된 문서)
├── Image Annotation: ON (이미지/다이어그램 포함 문서)
└── Gemini Enhancement: ON (복잡한 레이아웃, 차트, 다이어그램)
```

---

## 5. 청크 사이즈 설정

Layout Parser의 기본 청크 사이즈는 **500 토큰**이며, Google에서도 이 값을 권장합니다.

한 가지 주의할 점은 **Datastore 생성 후에는 청크 사이즈를 변경할 수 없다**는 것입니다. 변경하려면 새 Datastore를 생성해야 하며, 이 경우 인덱싱과 파싱 비용이 다시 발생합니다. 따라서 처음 설계 단계에서 청크 사이즈를 신중하게 결정하는 것이 중요합니다.

---

## 6. 할루시네이션 대응: Instruction 튜닝 팁

Layout Parser 옵션을 모두 켜더라도 **할루시네이션**(참고 소스 없이 그럴듯한 답변을 생성하는 현상)은 여전히 발생할 수 있습니다. 특히 참고 소스를 아무것도 표시하지 않은 답변의 경우, 대부분 틀린 정보였다는 실전 경험이 있습니다.

이를 완화하기 위한 Instruction 작성 팁입니다.

| 전략 | Instruction 예시 |
|------|-----------------|
| 소스 없으면 답변 거부 | "참고할 수 있는 문서가 없는 경우, 답변하지 마세요." |
| 오답의 위험성 강조 | "모르는 것에 대해 틀린 정보를 제공하는 것이 답변하지 않는 것보다 더 나쁩니다." |
| 불확실성 표현 요구 | "확실하지 않은 내용은 반드시 불확실하다고 표시하세요." |

핵심은 **"틀린 답변보다 모른다는 답변이 낫다"**는 원칙을 모델에 명확하게 전달하는 것입니다.

---

## 7. 정리

| 결정 포인트 | 권장 사항 |
|------------|----------|
| Parser 선택 | Layout Parser (Digital Parser 대비 정확도 2~3배 향상) |
| Table Annotation | ON |
| Image Annotation | ON |
| Gemini Enhancement | ON (Public Preview이지만 차트/다이어그램 이해에 필수적) |
| 청크 사이즈 | 500 (기본값, Google 권장) |
| 비용 증가 | 전체 옵션 활성화 시 토큰 비용 약 17% 증가 |
| 할루시네이션 대응 | Instruction에서 소스 없는 답변 거부 규칙 명시 |

ERP 매뉴얼처럼 복잡한 레이아웃의 문서를 다루는 RAG 시스템에서는 **"비싼 Parser + 모든 옵션 활성화"**가 상용 수준의 품질을 달성하기 위한 현실적인 선택입니다. 비용 증가분(약 17%)은 답변 정확도의 극적인 개선을 고려하면 충분히 정당화될 수 있습니다.
