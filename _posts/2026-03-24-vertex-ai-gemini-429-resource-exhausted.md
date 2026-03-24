---
layout: post
title: "Vertex AI Gemini API 429 RESOURCE_EXHAUSTED 에러 완전 정복"
date: 2026-03-24
tags: [vertex-ai, gemini, google-cloud, quota, rate-limit, troubleshooting, provisioned-throughput]
---

Vertex AI에서 Gemini API를 사용하다 보면 한 번쯤은 만나게 되는 에러가 있습니다.

```
429 RESOURCE_EXHAUSTED
```

"요청이 너무 많다"는 뜻인데, 단순히 기다린다고 해결되지 않는 경우도 있습니다. 이 에러는 **원인이 3가지**로 나뉘고, 각각 해결 방법이 다릅니다.

이 글에서는 429 에러의 원인별 분류, 실무 대응 패턴, 그리고 Provisioned Throughput(PT) 구매까지 포함한 전체 해결 전략을 정리합니다.

---

## 1. 429 에러의 3가지 원인

에러 메시지의 세부 내용을 보면 어떤 유형인지 구분할 수 있습니다.

### 1-1. Per-model, per-minute quota 초과 (RPM / TPM)

**가장 흔한 원인**입니다. 특정 모델에 대해 **분당 요청 수(RPM)** 또는 **분당 토큰 수(TPM)** 한도를 넘긴 경우입니다.

| 약어 | 의미 | 설명 |
|------|------|------|
| RPM | Requests Per Minute | 1분 동안 허용되는 API 호출 횟수 |
| TPM | Tokens Per Minute | 1분 동안 처리할 수 있는 총 토큰 수 |

모델별로 기본 quota가 다릅니다. 예를 들어 Gemini 2.5 Pro는 Gemini 2.0 Flash 대비 기본 RPM이 상당히 낮습니다.

**해결 방법:**

- Google Cloud Console → Vertex AI → Quotas에서 현재 할당량 확인
- 필요시 quota 상향 요청 (무료, 보통 1~2일 내 승인)
- 코드 레벨에서 exponential backoff + retry 적용
- 요청을 분산하거나 batch 처리로 전환

### 1-2. Concurrent request 제한 초과

동시에 너무 많은 요청이 **병렬로** 실행되는 경우입니다. 특히 streaming 요청이나 장시간 요청에서 자주 발생합니다.

**해결 방법:**

- 동시 요청 수를 제어하는 semaphore / concurrency limiter 적용
- `asyncio` 기반이면 `asyncio.Semaphore`로 동시 실행 수 제한

```python
import asyncio

semaphore = asyncio.Semaphore(5)  # 동시 요청 5개로 제한

async def call_gemini(prompt):
    async with semaphore:
        response = await client.aio.models.generate_content(
            model="gemini-2.5-pro",
            contents=prompt
        )
        return response
```

### 1-3. Region-level capacity 부족

Google 측 인프라의 **일시적 용량 부족**입니다. quota가 충분해도 발생할 수 있습니다.

**해결 방법:**

- 다른 region으로 fallback (예: `us-central1` → `europe-west1`)
- 잠시 후 재시도
- 사용자 측에서 근본적으로 해결할 수 없는 케이스이므로 retry 로직이 필수

> 참고로 `asia-northeast3`(서울)은 다른 region 대비 capacity가 상대적으로 제한적일 수 있습니다.

---

## 2. 분당 제한에 걸렸을 때 얼마나 기다려야 하나?

Per-minute 제한이므로, **최대 1분이면 풀립니다.**

Vertex AI의 RPM/TPM quota는 sliding window 또는 fixed 1-minute window 기반으로 동작합니다. 429를 받은 시점부터 해당 window가 리셋되면 다시 요청할 수 있으므로, 실질적으로 **수 초 ~ 최대 60초** 사이에 풀립니다.

### 실전 체감 대기 시간

| 상황 | 예상 대기 시간 |
|------|----------------|
| 한두 건 초과한 정도 | 10~20초 후 재시도로 통과 |
| 대량으로 한꺼번에 쏟아부은 경우 | window 리셋까지 약 60초 대기 |
| 연속 429를 맞으면서 계속 재시도 | 재시도 자체도 카운트에 포함되어 **오히려 더 지연** |

마지막 케이스가 중요합니다. 429를 받고 즉시 재시도하면 그 요청도 RPM에 카운트되어 악순환에 빠질 수 있습니다. **그래서 exponential backoff가 필수입니다.**

---

## 3. 실무 대응 패턴

### 3-1. Retry with Exponential Backoff

가장 기본적이면서 필수적인 패턴입니다.

```python
import time
import random
from google import genai

def call_with_retry(func, max_retries=5):
    for attempt in range(max_retries):
        try:
            return func()
        except genai.errors.ClientError as e:
            if "429" in str(e) and attempt < max_retries - 1:
                wait = (2 ** attempt) + random.uniform(0, 1)
                print(f"Rate limited. Retrying in {wait:.1f}s...")
                time.sleep(wait)
            else:
                raise
```

재시도 간격은 다음과 같이 점진적으로 증가합니다:

| 시도 | 대기 시간 (약) |
|------|----------------|
| 1차 재시도 | ~2초 |
| 2차 재시도 | ~4초 |
| 3차 재시도 | ~8초 |
| 4차 재시도 | ~16초 |

대부분 **3차 이내에 통과**됩니다. 무조건 1분을 통째로 기다릴 필요 없이, backoff로 점진적으로 재시도하면 빈 슬롯이 생기는 즉시 통과됩니다.

### 3-2. Quota 확인

현재 프로젝트의 Vertex AI quota를 확인하는 명령어입니다.

```bash
gcloud services quota list \
  --service=aiplatform.googleapis.com \
  --filter="metric:generateContent" \
  --format="table(metric,defaultLimit,usage)"
```

### 3-3. Quota 상향 요청

Console에서 직접 요청하거나 gcloud CLI로도 가능합니다.

**Console 경로:** IAM & Admin → Quotas → "GenerateContent" 검색 → Edit Quotas

```bash
gcloud alpha services quota update \
  --service=aiplatform.googleapis.com \
  --consumer=projects/YOUR_PROJECT_ID \
  --metric=aiplatform.googleapis.com/generate_content_requests_per_minute_per_project_per_base_model \
  --unit=1/min/{project} \
  --value=NEW_LIMIT
```

---

## 4. RAG + Chatbot 구조에서의 주의점

RAG(Retrieval-Augmented Generation) 기반 챗봇에서는 **한 번의 사용자 질의가 여러 번의 API 호출을 발생**시킵니다.

```
사용자 질의 1건
├── Retrieval (검색/임베딩) → API 호출 1회
├── Generation (응답 생성) → API 호출 1회
└── (필요시) Reranking/요약  → API 호출 추가
```

체감상 "몇 번 안 물어봤는데"라고 느껴도, 내부적으로는 RPM을 2~3배 속도로 소모하고 있을 수 있습니다.

Agent Builder나 ADK를 사용하는 경우에도 마찬가지입니다. Multi-turn call이 내부적으로 발생하면서 의도치 않게 RPM을 소진하는 케이스가 있으므로, **실제 API 호출 횟수를 모니터링**하는 것이 중요합니다.

---

## 5. Provisioned Throughput(PT) 구매로 해결할 수 있을까?

짧게 답하면, **거의 해결되지만 "완전히"는 아닙니다.**

### PT가 해결하는 것

| 문제 유형 | PT로 해결 여부 |
|-----------|---------------|
| Per-model quota 기반 429 | **해결** — 전용 처리 용량 보장 |
| Region-level capacity 부족 | **해결** — 다른 사용자와 용량 경쟁 없음 |
| 구매한 PT 용량 자체 초과 | **미해결** — 여전히 throttling 발생 |

PT를 구매하면 전용 처리 용량이 보장되므로, 일반적인 on-demand 환경의 429 에러는 사라집니다. 다른 사용자와 용량을 경쟁하지 않기 때문에, region-level capacity 부족 문제도 기본적으로 해소됩니다.

### 그래도 429가 발생하는 경우

구매한 PT 용량 자체를 초과하면 여전히 429가 발생합니다. 예를 들어 PT를 **초당 100 output token** 용량으로 구매했는데 실제 트래픽이 그 이상이면 동일하게 throttling됩니다.

> 결국 "quota의 주체가 Google 기본값 → 내가 구매한 용량"으로 바뀌는 것이지, 무제한이 되는 건 아닙니다.

### PT 구매 전 체크포인트

**1. 비용 구조 이해**

PT는 사용 여부와 무관하게 **예약한 용량만큼 과금**됩니다. PoC 단계에서는 트래픽이 불규칙하므로 비용 효율이 낮을 수 있습니다.

**2. Quota 상향 요청이 먼저**

대부분의 429는 기본 quota가 낮아서 발생하는 것이라, Console에서 quota increase request만 해도 해결되는 경우가 많습니다. **이건 무료입니다.**

**3. 실제 필요 용량 산정**

동시 사용자 수 × 질의당 API 호출 횟수 × 평균 토큰 수를 먼저 계산해보고, 그게 현재 기본 quota를 넘는지 확인하는 게 우선입니다.

---

## 6. 추천 해결 순서

429 에러를 만났을 때, 다음 순서로 대응하는 것을 권장합니다.

```
Step 1. 현재 quota 확인
        └── 어떤 모델, 어떤 region에서 발생하는지 파악

Step 2. 기본 quota 상향 요청 (무료)
        └── Console에서 요청, 보통 1~2일 내 승인

Step 3. 코드에 retry + exponential backoff 적용
        └── 일시적인 429는 대부분 이것으로 해결

Step 4. 동시 요청 수 제어
        └── Semaphore 등으로 concurrency 제한

Step 5. 그래도 부족하면 Provisioned Throughput 검토
        └── 안정적인 SLA가 필요한 프로덕션 환경에서 고려
```

---

## 빠른 체크리스트

429 에러가 발생했을 때 즉시 확인할 포인트입니다.

- [ ] **어떤 모델에서 발생하는가?** — Gemini 2.5 Pro는 기본 RPM이 낮음
- [ ] **어떤 region을 사용 중인가?** — `asia-northeast3`는 capacity가 제한적일 수 있음
- [ ] **내부적으로 multi-turn call이 RPM을 소진하고 있지 않은가?** — Agent Builder, ADK 사용 시 확인
- [ ] **여러 사람이 동시에 테스트 중인가?** — PoC 환경에서 흔한 케이스
- [ ] **RAG 구조에서 한 질의당 실제 API 호출 횟수는?** — retrieval + generation으로 2회 이상일 수 있음
- [ ] **retry 로직이 적용되어 있는가?** — exponential backoff 없는 즉시 재시도는 오히려 악화

---

## 마무리

429 RESOURCE_EXHAUSTED는 Vertex AI Gemini API를 사용하면서 피할 수 없는 에러입니다. 하지만 원인을 정확히 파악하면 대부분 간단하게 해결할 수 있습니다.

**대부분의 경우 quota 상향 요청 + exponential backoff 조합이면 충분합니다.** PT는 프로덕션 수준의 안정성이 필요할 때 비로소 검토하면 됩니다.
