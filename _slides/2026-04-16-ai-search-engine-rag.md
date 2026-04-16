---
title: "AI와 검색 엔진 — RAG를 위한 검색 기술 완전 가이드"
date: 2026-04-16
tags: [information-retrieval, sparse-search, dense-search, bm25, cosine-similarity, jaccard, tanimoto, korean-morphology, nori, mecab, hybrid-search, elasticsearch, opensearch, qdrant, weaviate, milvus, vertex-ai-search, rag, llm]
description: "IR의 역사부터 Jaccard·Tanimoto·Cosine 유사도, BM25, Hybrid Search, 오픈소스 검색 엔진, Google Vertex AI Search까지 — AI LLM RAG를 위한 검색 기술 완전 가이드."
theme: white
---

# AI와 검색 엔진

Sparse · Dense · Hybrid — RAG를 위한 검색 기술 완전 가이드

[seonghak.com/slides/2026-04-16-ai-search-engine-rag/](https://seonghak.com/slides/2026-04-16-ai-search-engine-rag/)

---

## 목차

1. **IR의 역사** — 정보 검색의 탄생과 진화
2. **Sparse Search** & BM25
3. **한국어 형태소 분석** — Sparse Search의 핵심 전처리
4. **유사도 알고리즘** — Cosine, Jaccard, Tanimoto 등
5. **Dense Search** & Vector Search
6. **Hybrid Search** 전략 & RRF
7. **오픈소스 하이브리드 검색 엔진**
8. **Google Hybrid Search** 기술
9. AI LLM **RAG**를 위한 검색엔진 활용
10. 정리 & Q&A

---

## 검색의 진화

| 세대 | 방식 | 핵심 기술 | 한계 |
|------|------|----------|------|
| **1세대** | 키워드 매칭 | Boolean, TF-IDF | 동의어·문맥 무시 |
| **2세대** | 통계 기반 랭킹 | **BM25** | 의미 유사성 미반영 |
| **3세대** | 의미 기반 검색 | **Embedding + ANN** | 키워드 정확도 ↓ |
| **4세대** | 하이브리드 + AI | **Hybrid Search + LLM** | 현재 최적 해법 |

> 키워드 → 통계 → 벡터 → **하이브리드** 로 진화

---

## IR(Information Retrieval)의 역사

**정보 검색**은 컴퓨터 이전부터 시작된 학문 분야

| 연대 | 사건 | 의의 |
|------|------|------|
| **1945** | Vannevar Bush — *"As We May Think"* | 하이퍼텍스트·정보 검색 개념의 시초 |
| **1950s** | Calvin Mooers — "Information Retrieval" 용어 최초 사용 | IR 학문 분야 탄생 |
| **1960s** | Gerard Salton — **SMART 시스템** | TF-IDF, 벡터 공간 모델(VSM) 정립 |
| **1970s** | Robertson & Spärck Jones — **확률 모델** | IDF 개념 공식화, 확률적 랭킹 원리 |
| **1992** | TREC (Text REtrieval Conference) 시작 | IR 평가 표준화, 벤치마크 확립 |
| **1994** | Robertson et al. — **BM25** 발표 | Okapi BM25, 현재까지 표준 |
| **1998** | Google — **PageRank** | 링크 구조 기반 웹 검색 혁명 |
| **2013** | Word2Vec (Mikolov) | 단어 임베딩의 시작 |
| **2018** | BERT (Devlin et al.) | 문맥 기반 임베딩 → Dense Retrieval 기반 |
| **2020~** | DPR, ColBERT, Hybrid Search | 밀집 검색 + 하이브리드 시대 개막 |

---

## IR의 핵심 개념 — 벡터 공간 모델 (VSM)

**Gerard Salton (1971)** 이 정립한 IR의 수학적 토대

### 핵심 아이디어

> 문서와 쿼리를 **같은 벡터 공간의 점**으로 표현하고, **거리/각도**로 유사도를 측정한다

```
            단어2 ↑
                 │    · 문서A
                 │   /
                 │  / θ (코사인 각도)
                 │ /
                 │/________· 쿼리Q ──→ 단어1
```

### VSM이 열어준 세계

| 이전 | VSM 이후 |
|------|----------|
| Boolean 검색 (있다/없다) | **유사도 점수**로 랭킹 |
| 결과 = 집합 | 결과 = **정렬된 리스트** |
| 부분 매치 불가 | **부분 매치 + 가중치** |

> 오늘날 Sparse Search와 Dense Search **모두** VSM의 후손

---

## Sparse Search란?

**단어 존재 여부**를 기반으로 문서를 매칭하는 전통적 검색 방식

- 문서를 **고차원 희소 벡터**(Sparse Vector)로 표현
- 대부분의 값이 **0**, 해당 단어가 있는 차원만 **비-0**
- 대표 알고리즘: **TF-IDF**, **BM25**

```
문서: "검색 엔진은 AI 기술이다"

벡터: [0, 0, 1, 0, 0, 1, 0, 0, 1, 0, 0, 1, 0, ...]
       ↑        ↑        ↑        ↑
      검색     엔진      AI      기술
```

> 어휘 수 = 벡터 차원 수 → **수만~수십만 차원**, 대부분 0

---

## BM25 (Best Matching 25)

Sparse Search의 **사실상 표준 랭킹 함수**

### 핵심 공식

**score(D, Q) = Σ IDF(qi) · [ f(qi, D) · (k1 + 1) ] / [ f(qi, D) + k1 · (1 - b + b · \|D\| / avgdl) ]**

### 구성 요소

| 요소 | 의미 | 역할 |
|------|------|------|
| **IDF(qi)** | 역문서 빈도 | 희귀한 단어일수록 높은 점수 |
| **f(qi, D)** | 단어 빈도 (TF) | 문서 내 등장 횟수 |
| **k1** | TF 포화 파라미터 | 반복 등장의 감쇠율 (기본 1.2) |
| **b** | 문서 길이 보정 | 긴 문서 패널티 (기본 0.75) |
| **avgdl** | 평균 문서 길이 | 정규화 기준 |

> TF-IDF의 진화형 — **TF 포화** + **문서 길이 보정** 추가

---

## BM25의 장단점

### 장점

- **빠르다** — 역색인(Inverted Index) 기반, 밀리초 단위
- **설명 가능** — 어떤 키워드가 매칭되었는지 추적 가능
- **제로 학습** — 별도 모델 훈련 불필요
- **업계 표준** — Elasticsearch, OpenSearch, Solr 등 모든 검색엔진 내장

### 단점

- **어휘 불일치(Vocabulary Mismatch)** — "자동차" 검색 시 "차량" 미매칭
- **문맥 무시** — "사과"가 과일인지 사죄인지 구분 불가
- **의미적 유사 문서 놓침** — 같은 의미 다른 표현 검색 불가

---

## 한국어와 Sparse Search — 왜 어려운가?

영어와 달리 한국어는 **띄어쓰기만으로 단어를 분리할 수 없다**

### 영어 vs 한국어 토큰화

```
영어: "I love search engines"
  → Whitespace 분리 → ["I", "love", "search", "engines"]
  → 바로 BM25 적용 가능 ✓

한국어: "검색엔진을 사랑합니다"
  → Whitespace 분리 → ["검색엔진을", "사랑합니다"]
  → "검색엔진" 검색 시 "검색엔진을"과 미매칭 ✗
```

### 한국어의 특수성

| 특성 | 영어 | 한국어 |
|------|------|--------|
| **어순** | SVO (고정적) | SOV (유연) |
| **조사** | 전치사 (별도 단어) | 조사가 단어에 **붙음** |
| **활용** | 어미 변화 적음 | 동사/형용사 **어미가 수십 가지** |
| **복합어** | 띄어쓰기로 구분 | **붙여쓰기** 빈번 |
| **띄어쓰기** | 규칙적 | **불규칙** (오류 다수) |

> 형태소 분석 없이는 Sparse Search **정확도가 크게 하락**

---

## 형태소 분석기 (Morphological Analyzer)

한국어 문장을 **의미 있는 최소 단위(형태소)**로 분리하는 도구

### 분석 예시

```
입력: "검색엔진을 사랑합니다"

형태소 분석 결과:
  검색  / NNG (일반명사)
  엔진  / NNG (일반명사)
  을    / JKO (목적격 조사)
  사랑  / NNG (일반명사)
  하    / XSV (동사 파생 접미사)
  ㅂ니다 / EF (종결 어미)
```

### 왜 필요한가?

| 분석 전 | 분석 후 | 효과 |
|---------|---------|------|
| "검색엔진을" | "검색" + "엔진" | **복합어 분리** |
| "사랑합니다" | "사랑" + "하다" | **어간 추출** |
| "달리고 있는" | "달리" + "고 있" | **활용 정규화** |

> 형태소 분석 = Sparse Search의 **한국어 전처리 필수 단계**

---

## 주요 한국어 형태소 분석기

| 분석기 | 개발 | 라이선스 | 특징 |
|--------|------|---------|------|
| **Nori** | Elastic (루씬) | Apache 2.0 | ES/OpenSearch **내장**, mecab-ko 사전 기반 |
| **Mecab-ko** | 은전한닢 | GPL/LGPL/BSD | 가장 빠름, C++ 기반, 사전 커스터마이징 |
| **Komoran** | Shineware | Apache 2.0 | Java 네이티브, 외부 라이브러리 불필요 |
| **Hannanum** | KAIST | GPL | 학술 목적, 오래된 사전 |
| **Kkma** | 서울대 IDS | GPL | 정확하지만 느림 |
| **Okt (Open Korean Text)** | Twitter → 커뮤니티 | Apache 2.0 | 정규화·어구 추출, 비공식 텍스트에 강함 |
| **Kiwi** | bab2min | LGPL | 최신, Rust/C++ 기반, 빠르고 정확 |

> Elasticsearch/OpenSearch 사용 시 → **Nori** 가 사실상 표준

---

## Nori 분석기 — Elasticsearch 한국어 검색

### 설정 예시

```json
{
  "settings": {
    "analysis": {
      "tokenizer": {
        "nori_custom": {
          "type": "nori_tokenizer",
          "decompound_mode": "mixed",
          "user_dictionary": "userdict_ko.txt"
        }
      },
      "analyzer": {
        "korean": {
          "type": "custom",
          "tokenizer": "nori_custom",
          "filter": [
            "nori_readingform",
            "nori_part_of_speech",
            "lowercase"
          ]
        }
      }
    }
  }
}
```

### decompound_mode 비교

| 모드 | "삼성전자" 결과 | 용도 |
|------|----------------|------|
| **none** | ["삼성전자"] | 복합어 유지 |
| **discard** | ["삼성", "전자"] | 분리만 |
| **mixed** | ["삼성전자", "삼성", "전자"] | **추천** — 원본 + 분리 모두 색인 |

---

## 한국어 Sparse Search의 함정들

### 1. 띄어쓰기 오류

```
"머신러닝" vs "머신 러닝" vs "머신런닝"
→ 형태소 분석기에 따라 다르게 토큰화 → BM25 점수 달라짐
```

### 2. 동음이의어

```
"배" → 과일? 선박? 신체부위? 배수?
→ 형태소 분석기는 문맥 구별 불가
```

### 3. 신조어·전문 용어

```
"쿠버네티스", "RAG", "LangChain" → 사전에 없으면 분석 실패
→ 사용자 사전(user_dictionary) 관리 필수
```

### 4. 조사/어미 변이

```
"검색하다" / "검색한다" / "검색했다" / "검색할" / "검색하는"
→ 모두 "검색" + "하다" 로 정규화해야 올바른 매칭
```

> **사용자 사전 관리**와 **분석기 선택**이 한국어 검색 품질을 결정

---

## 한국어 검색 — Sparse의 한계, 해법은?

### Sparse만으로 부족한 이유

| 문제 | 예시 | Sparse 결과 |
|------|------|------------|
| **동의어** | "노트북" vs "랩톱" | 미매칭 |
| **약어** | "쿠버" vs "쿠버네티스" | 미매칭 |
| **한영 혼용** | "AI 검색" vs "인공지능 검색" | 부분 매칭 |
| **오타** | "엘라스틱서치" vs "일래스틱서치" | 미매칭 |
| **의미 검색** | "서버가 느려요" → 성능 최적화 문서 | 키워드 없어 미매칭 |

### 해법 조합

| 계층 | 전략 | 효과 |
|------|------|------|
| **전처리** | Nori + 사용자 사전 + 동의어 사전 | 기본 품질 확보 |
| **검색** | **Hybrid Search** (Sparse + Dense) | 의미 매칭 보완 |
| **후처리** | Re-ranking (Cross-encoder / LLM) | 최종 정밀도 향상 |

> 한국어 RAG에서는 **형태소 분석 + Hybrid Search**가 필수 조합

---

## Dense Search란?

**의미(Semantic)**를 기반으로 문서를 매칭하는 검색 방식

- 문서와 쿼리를 **저차원 밀집 벡터**(Dense Vector)로 변환
- **모든** 차원에 의미 있는 값이 들어감 (768~1536차원)
- 사전 학습된 **Embedding 모델**이 벡터 생성

```
"검색 엔진은 AI 기술이다"
  → Embedding Model
  → [0.23, -0.41, 0.87, 0.12, -0.65, 0.33, ...]
     (768 또는 1536 차원의 밀집 벡터)
```

> 단어가 아닌 **의미**를 수치화 → 동의어/유사 표현 검색 가능

---

## Cosine Similarity (코사인 유사도)

Dense Search에서 **두 벡터의 유사도를 측정**하는 핵심 지표

### 공식

**cos(θ) = (A · B) / (\|\|A\|\| × \|\|B\|\|)**

| 값 | 의미 |
|----|------|
| **1.0** | 완전히 같은 방향 (매우 유사) |
| **0.0** | 직교 (관련 없음) |
| **-1.0** | 반대 방향 (반대 의미) |

### 왜 코사인인가?

- 벡터의 **크기(문서 길이)**에 영향 받지 않음
- 오직 **방향(의미)**만 비교
- 고차원 공간에서 효율적으로 계산 가능

---

## 유사도 알고리즘 총정리

코사인 유사도 외에도 IR에서 사용되는 다양한 유사도 측정 방법

| 알고리즘 | 입력 타입 | 값 범위 | 대표 사용처 |
|----------|----------|---------|------------|
| **Cosine Similarity** | 실수 벡터 | -1 ~ 1 | Dense/Sparse 검색 |
| **Jaccard 계수** | 집합 | 0 ~ 1 | 키워드 집합 비교 |
| **Tanimoto 계수** | 이진/실수 벡터 | 0 ~ 1 | 화학·지문 유사도 |
| **유클리드 거리** | 실수 벡터 | 0 ~ ∞ | kNN, 클러스터링 |
| **내적 (Dot Product)** | 실수 벡터 | -∞ ~ ∞ | MIPS 기반 검색 |
| **Manhattan 거리** | 실수 벡터 | 0 ~ ∞ | 희소 데이터 |
| **Hamming 거리** | 이진 벡터 | 0 ~ n | 해시 기반 검색 |

---

## Jaccard 계수 (Jaccard Coefficient)

**두 집합의 겹침 정도**를 측정하는 가장 직관적인 유사도

### 공식

**J(A, B) = \|A ∩ B\| / \|A ∪ B\|**

### 예시

```
문서A 키워드: {AI, 검색, 엔진, 기술}
문서B 키워드: {AI, 머신러닝, 기술, 딥러닝}

교집합: {AI, 기술}           → |A ∩ B| = 2
합집합: {AI, 검색, 엔진, 기술, 머신러닝, 딥러닝} → |A ∪ B| = 6

J(A, B) = 2 / 6 = 0.33
```

### 특징

- **장점**: 단순·직관적, 집합 크기에 정규화됨
- **단점**: 단어 빈도(TF) 무시, 이진(있다/없다)만 고려
- **활용**: 문서 중복 탐지, 쿼리 확장, MinHash(LSH)

---

## Tanimoto 계수 (Tanimoto Coefficient)

Jaccard 계수를 **연속 값 벡터**로 확장한 유사도

### 공식

**T(A, B) = (A · B) / (\|\|A\|\|² + \|\|B\|\|² - A · B)**

### Jaccard vs Tanimoto

| 구분 | Jaccard | Tanimoto |
|------|---------|----------|
| **입력** | 집합 (이진) | 벡터 (실수/이진) |
| **값 범위** | 0 ~ 1 | 0 ~ 1 |
| **TF 반영** | ✗ 불가 | ✓ 가능 |
| **Jaccard와 관계** | - | 이진 벡터일 때 Jaccard = Tanimoto |

### 활용 분야

- **화학 정보학** — 분자 지문(Fingerprint) 유사도 비교
- **추천 시스템** — 사용자 선호도 벡터 비교
- **문서 유사도** — TF 가중 키워드 벡터 비교

---

## 유클리드 거리 vs 코사인 유사도

**거리** vs **방향** — 무엇을 측정하느냐의 차이

### 유클리드 거리

**d(A, B) = √(Σ(ai - bi)²)**

### 비교

```
       ↑ 차원2
       │         · B (0.8, 0.9)
       │        /|
       │       / |  ← 유클리드 거리 (직선)
       │      /  |
       │   · A (0.2, 0.3)
       │  θ ← 코사인 각도
       └──────────→ 차원1
```

| 상황 | 코사인 유사도 | 유클리드 거리 |
|------|-------------|-------------|
| 같은 방향, 다른 크기 | **1.0** (동일) | **크다** (다름) |
| 다른 방향, 같은 크기 | **낮다** (다름) | **작을 수 있음** |

> IR에서는 **코사인 유사도**가 표준 — 문서 길이 영향을 제거하기 때문

---

## 유사도 알고리즘 선택 가이드

| 상황 | 추천 알고리즘 | 이유 |
|------|-------------|------|
| **Dense 벡터 검색** | Cosine Similarity | 방향 기반, 정규화된 비교 |
| **Sparse 키워드 비교** | Jaccard / BM25 | 집합 매칭 / 가중 랭킹 |
| **이진 특성 비교** | Tanimoto / Hamming | 이진 벡터 특화 |
| **kNN / 클러스터링** | 유클리드 거리 | 절대적 거리 필요 시 |
| **대규모 ANN 검색** | 내적 (Dot Product) | MIPS 최적화, 속도 우선 |
| **문서 중복 탐지** | Jaccard + MinHash | LSH 기반 확장 가능 |

> 정답은 없다 — **데이터 특성과 사용 목적**에 따라 선택

---

## Text Search의 Cosine Similarity

**TF-IDF 벡터** 간의 코사인 유사도 계산

```
문서1: "AI 검색 엔진"    → TF-IDF 벡터 A = [0.5, 0.3, 0.4, 0, 0, ...]
문서2: "검색 기술 활용"  → TF-IDF 벡터 B = [0, 0.3, 0, 0.6, 0.2, ...]

cos(A, B) = (0×0 + 0.3×0.3 + ...) / (||A|| × ||B||) = 0.12
```

### 특징

| 항목 | 설명 |
|------|------|
| **벡터 차원** | 어휘 크기 (수만~수십만) |
| **벡터 타입** | Sparse (대부분 0) |
| **유사도 의미** | 공통 키워드 비율 |
| **한계** | 다른 단어 = 유사도 0 (의미 무시) |

> "자동차"와 "차량"은 다른 차원 → **cos = 0** (유사하지 않다고 판단)

---

## Vector Search의 Cosine Similarity

**Embedding 벡터** 간의 코사인 유사도 계산

```
쿼리:  "자동차 엔진 문제"  → [0.82, -0.31, 0.45, ...]
문서1: "차량 모터 고장"    → [0.79, -0.28, 0.51, ...]  cos = 0.96
문서2: "비행기 엔진 설계"  → [0.21, 0.67, -0.42, ...]  cos = 0.34
```

### 특징

| 항목 | 설명 |
|------|------|
| **벡터 차원** | 768 ~ 1536 (고정) |
| **벡터 타입** | Dense (모든 값 비-0) |
| **유사도 의미** | **의미적** 유사도 |
| **강점** | 동의어/패러프레이즈 매칭 가능 |

> "자동차"와 "차량"은 유사한 벡터 → **cos ≈ 0.95** (유사하다고 판단)

---

## Text Search vs Vector Search 비교

| 비교 항목 | Text Search (Sparse) | Vector Search (Dense) |
|----------|---------------------|----------------------|
| **표현 방식** | TF-IDF / BM25 희소 벡터 | Embedding 밀집 벡터 |
| **벡터 차원** | 어휘 수 (수만~수십만) | 768 ~ 1536 |
| **유사도 기준** | 공통 키워드 존재 | 의미적 거리 |
| **동의어 처리** | ✗ 불가 | ✓ 자동 처리 |
| **정확한 키워드** | ✓ 완벽 매칭 | △ 놓칠 수 있음 |
| **속도** | ✓ 매우 빠름 | △ ANN 인덱스 필요 |
| **모델 의존성** | ✗ 없음 | ✓ Embedding 모델 필수 |
| **설명 가능성** | ✓ 높음 | ✗ 블랙박스 |

> **둘 다 장단점이 뚜렷** → 결합하면?

---

## Hybrid Search란?

**Sparse + Dense를 결합**하여 양쪽 장점을 취하는 검색 전략

### 기본 구조

```
         ┌─────────────────┐
 쿼리 ──→│  Sparse Search   │──→ BM25 점수
         │  (키워드 매칭)    │
         └─────────────────┘
                                  ──→ 결합(Fusion) ──→ 최종 랭킹
         ┌─────────────────┐
 쿼리 ──→│  Dense Search    │──→ 코사인 유사도
         │  (의미 매칭)      │
         └─────────────────┘
```

### 결합 방법

| 방법 | 설명 |
|------|------|
| **가중 합산** | `α × BM25 + (1-α) × Cosine` |
| **RRF (Reciprocal Rank Fusion)** | 순위 기반 결합, 점수 스케일 무관 |
| **Cross-encoder Re-ranking** | 후보군 추출 후 LLM으로 재랭킹 |

---

## Hybrid Search의 효과

| 쿼리 | Sparse Only | Dense Only | **Hybrid** |
|------|------------|-----------|-----------|
| "GCP VPC 방화벽 설정" | ✓ 정확 매칭 | △ 유사 결과 혼입 | **✓ 정확 + 관련 문서** |
| "클라우드 보안 강화 방법" | △ 키워드 없으면 누락 | ✓ 의미 매칭 | **✓ 폭넓게 커버** |
| "k8s pod OOMKilled" | ✓ 전문 용어 매칭 | △ 일반화 | **✓ 정확 + 맥락** |
| "서버 느려졌어요" | △ 모호한 키워드 | ✓ 의도 파악 | **✓ 최적 결과** |

> Hybrid = **정확성(Sparse)** + **재현율(Dense)** 의 최적 조합

---

## RRF (Reciprocal Rank Fusion)

가장 널리 쓰이는 하이브리드 결합 알고리즘

### 공식

**RRF_score(d) = Σ 1 / (k + rank_i(d))**

- `k` = 상수 (보통 60)
- `rank_i(d)` = i번째 검색 방식에서의 문서 d 순위

### 예시

| 문서 | BM25 순위 | Dense 순위 | RRF 점수 |
|------|----------|-----------|---------|
| A | 1 | 3 | 1/61 + 1/63 = **0.032** |
| B | 3 | 1 | 1/63 + 1/61 = **0.032** |
| C | 2 | 2 | 1/62 + 1/62 = **0.032** |
| D | 5 | 10 | 1/65 + 1/70 = **0.030** |

> 점수 스케일이 달라도 **순위 기반**으로 공정하게 결합

---

## 오픈소스 하이브리드 검색 엔진

Hybrid Search를 지원하는 주요 오픈소스 검색 엔진

| 엔진 | 라이선스 | Sparse | Dense | Hybrid | 특징 |
|------|---------|--------|-------|--------|------|
| **Elasticsearch** | SSPL / ELv2 | ✓ BM25 | ✓ kNN | ✓ RRF | 가장 넓은 생태계, 8.x부터 벡터 검색 |
| **OpenSearch** | Apache 2.0 | ✓ BM25 | ✓ kNN | ✓ 가중합 | AWS 주도, ES 포크, 완전 오픈소스 |
| **Weaviate** | BSD-3 | ✓ BM25 | ✓ HNSW | ✓ 네이티브 | 벡터 네이티브 DB, GraphQL API |
| **Qdrant** | Apache 2.0 | ✓ Sparse | ✓ HNSW | ✓ Fusion | Rust 기반, 고성능, gRPC/REST |
| **Milvus** | Apache 2.0 | ✓ Sparse | ✓ 다수 | ✓ RRF | 대규모 벡터 검색 특화, GPU 지원 |
| **Vespa** | Apache 2.0 | ✓ BM25 | ✓ ANN | ✓ 네이티브 | Yahoo 출신, ML 랭킹 내장 |
| **Apache Solr** | Apache 2.0 | ✓ BM25 | ✓ kNN | △ 수동 | 전통 강자, 9.x부터 벡터 |

---

## 오픈소스 엔진 — 계보와 선택 기준

### 두 가지 계보

```
전통 검색 엔진 (Sparse 출발)          벡터 DB (Dense 출발)
─────────────────────────         ──────────────────────
Elasticsearch (2010)               Milvus (2019)
OpenSearch (2021, ES 포크)          Qdrant (2021)
Apache Solr (2004)                 Weaviate (2019)
Vespa (2017, 오픈소스)              Chroma (2022)

         ───→  양쪽 모두 Hybrid로 수렴 중  ←───
```

### 선택 기준

| 기준 | 추천 |
|------|------|
| **기존 ES/Solr 운영 중** | Elasticsearch 8.x / OpenSearch 업그레이드 |
| **벡터 검색 중심 신규** | Qdrant / Weaviate |
| **대규모 (억 건+)** | Milvus / Vespa |
| **완전 오픈소스 필수** | OpenSearch / Qdrant / Milvus |
| **관리형 원하면** | Vertex AI Search / Amazon Kendra |

---

## Elasticsearch 8.x — Hybrid Search 예시

```json
{
  "retriever": {
    "rrf": {
      "retrievers": [
        {
          "standard": {
            "query": {
              "match": {
                "content": "클라우드 보안 강화"
              }
            }
          }
        },
        {
          "knn": {
            "field": "content_vector",
            "query_vector_builder": {
              "text_embedding": {
                "model_id": "my-embedding-model",
                "model_text": "클라우드 보안 강화"
              }
            },
            "k": 10,
            "num_candidates": 100
          }
        }
      ],
      "rank_window_size": 50,
      "rank_constant": 60
    }
  }
}
```

> ES 8.x부터 **RRF 기반 하이브리드 검색**을 네이티브 지원

---

## Qdrant — Hybrid Search 예시

```python
from qdrant_client import QdrantClient, models

client = QdrantClient("localhost", port=6333)

results = client.query_points(
    collection_name="documents",
    prefetch=[
        models.Prefetch(
            query=models.SparseVector(
                indices=[1, 42, 1337],
                values=[0.5, 0.8, 0.6]
            ),
            using="bm25-sparse",
            limit=20,
        ),
        models.Prefetch(
            query=dense_embedding,  # [0.23, -0.41, ...]
            using="dense-vector",
            limit=20,
        ),
    ],
    query=models.FusionQuery(
        fusion=models.Fusion.RRF
    ),
    limit=10,
)
```

> Qdrant는 **Sparse + Dense 벡터를 동시 저장**하고 RRF로 결합

---

## Google Vertex AI Search 아키텍처

Google Cloud의 **엔터프라이즈 검색 서비스**

### 핵심 구성

| 컴포넌트 | 역할 |
|----------|------|
| **Data Store** | 문서 저장소 (웹, Cloud Storage, BigQuery 등) |
| **Indexing** | 자동 Sparse + Dense 인덱스 생성 |
| **Search Engine** | 하이브리드 검색 + 랭킹 |
| **Serving Config** | 부스팅, 필터, 스니펫 설정 |

### 지원 데이터 소스

- Unstructured (PDF, HTML, TXT)
- Structured (JSON, BigQuery)
- Website (크롤링)
- Healthcare (FHIR)

---

## Google Hybrid Search 기술

Vertex AI Search는 **자동으로 Hybrid Search를 수행**

### Google의 접근 방식

| 단계 | 기술 | 설명 |
|------|------|------|
| **1. Sparse** | 자체 BM25 변형 | 키워드 기반 후보 추출 |
| **2. Dense** | Gecko / 자체 Embedding | 의미 기반 후보 추출 |
| **3. Fusion** | Google 자체 알고리즘 | Sparse + Dense 결합 |
| **4. Re-ranking** | LLM 기반 Re-ranker | 최종 관련성 정렬 |
| **5. Summarization** | Gemini | 검색 결과 요약·답변 |

### 차별점

- **자동 Chunking** — 문서를 의미 단위로 자동 분할
- **Layout Parser** — PDF 표/그림 구조 인식
- **Extractive Answer** — 원문에서 정답 구간 추출
- **Grounding** — 답변의 출처(Citation) 자동 제공

---

## Vertex AI Search 설정 옵션

| 설정 | 설명 | 기본값 |
|------|------|--------|
| **search_type** | SEMANTIC / KEYWORD / HYBRID | HYBRID |
| **boost_spec** | 특정 필드/값 부스팅 | 없음 |
| **filter** | 메타데이터 필터링 | 없음 |
| **snippet_spec** | 스니펫 반환 여부 | 활성 |
| **content_search_spec** | 요약/추출 답변 설정 | 비활성 |
| **chunk_spec** | 청크 크기/오버랩 | 자동 |

> `search_type: HYBRID`가 기본 — **별도 설정 없이 하이브리드 검색**

---

## RAG (Retrieval-Augmented Generation)

LLM의 한계를 **검색(Retrieval)**으로 보완하는 아키텍처

### LLM의 한계

- **할루시네이션** — 없는 정보를 생성
- **지식 컷오프** — 학습 이후 정보 모름
- **내부 데이터 부재** — 회사 문서 접근 불가

### RAG 파이프라인

```
질문 → [1. Retrieval] → 관련 문서 검색
                ↓
     → [2. Augment]  → 프롬프트에 문서 삽입
                ↓
     → [3. Generate] → LLM이 문서 기반 답변 생성
```

> **검색 품질 = RAG 품질** — 검색이 틀리면 답변도 틀린다

---

## RAG에서 검색 전략 비교

| 전략 | RAG 적합도 | 이유 |
|------|-----------|------|
| **Sparse Only** | △ | 키워드 정확하지만 의미 누락 |
| **Dense Only** | △ | 의미 캡처하지만 고유명사 약함 |
| **Hybrid** | ✓✓ | 정확성 + 재현율 모두 확보 |
| **Hybrid + Re-rank** | ✓✓✓ | 최종 관련성까지 최적화 |

### RAG 검색의 핵심 지표

| 지표 | 의미 | 검색 전략 영향 |
|------|------|---------------|
| **Recall@K** | 상위 K개에 정답 포함 비율 | Hybrid > Dense > Sparse |
| **Precision@K** | 상위 K개 중 정답 비율 | Re-rank 적용 시 ↑ |
| **MRR** | 첫 정답의 역순위 평균 | Hybrid + Re-rank 최적 |

---

## RAG + Google Vertex AI Search

### 구현 아키텍처

```
사용자 질문
    ↓
[Vertex AI Search]  ← 하이브리드 검색 + Re-ranking
    ↓
관련 청크 (Top-K)
    ↓
[Prompt 구성]  ← 시스템 프롬프트 + 검색 결과 + 사용자 질문
    ↓
[Gemini API]   ← 문서 기반 답변 생성 + Citation
    ↓
최종 응답 (답변 + 출처)
```

### Vertex AI Search가 RAG에 적합한 이유

- **자동 하이브리드 검색** — 별도 설정 없이 Sparse + Dense
- **자동 청킹** — 문서를 의미 단위로 분할
- **Grounding API** — 답변에 출처(Citation) 자동 부여
- **멀티모달** — PDF 표/그림 내용도 검색 가능
- **관리형 서비스** — 인프라 관리 불필요

---

## 실전 Tip: RAG 검색 품질 높이기

### 1. 문서 전처리

- 의미 단위 **청킹** (500~1000 토큰 권장)
- **메타데이터** 추가 (카테고리, 날짜, 소스)
- 표/그림은 **텍스트 설명** 추가

### 2. 검색 최적화

- **Hybrid Search** 기본 활성화
- Boost 규칙으로 최신/중요 문서 **가중치 부여**
- **필터**로 불필요한 문서 제외

### 3. 후처리

- **Re-ranking** 적용 (Cross-encoder 또는 LLM 기반)
- 답변에 반드시 **출처(Citation)** 포함
- **Hallucination 탐지** — 검색 결과와 답변 대조

---

## 전체 기술 스택 정리

| 계층 | 기술 | 역할 |
|------|------|------|
| **색인** | Inverted Index + Vector Index | Sparse + Dense 저장 |
| **검색** | BM25 + Embedding + ANN | 후보 문서 검색 |
| **결합** | RRF / 가중 합산 | 하이브리드 점수 산출 |
| **재랭킹** | Cross-encoder / LLM | 최종 관련성 정렬 |
| **생성** | Gemini / GPT / Claude | 문서 기반 답변 생성 |
| **검증** | Grounding / Citation | 출처 확인 + 할루시네이션 방지 |

> 검색은 RAG의 **기초 체력** — 검색이 정확해야 AI가 똑똑해진다

---

# 감사합니다

AI와 검색 엔진 — RAG를 위한 검색 기술 가이드

문의: [tech.mzgcp.net](https://tech.mzgcp.net)
