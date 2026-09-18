---
layout: post
title: "Gemini Search Grounding을 이용해서 뉴스 크롤러 만들기"
date: 2026-05-29
tags: [gemini, search-grounding, crawler, gcp, news]
---

"특정 키워드에 대한 뉴스를 매일 자동으로 모으고 싶다."

흔한 요구입니다. 우리 회사 언급 기사, 경쟁사 동향, 특정 산업의 신제품 소식. 보통은 뉴스 크롤러를 떠올립니다. 그런데 막상 만들어 보면 손이 많이 갑니다. 언론사마다 HTML 구조가 다르고, JavaScript로 본문을 렌더링하는 곳도 있고, 봇 차단을 피해야 하고, 무엇보다 **어떤 언론사를 긁을지부터 정해야 합니다.** 네이버 뉴스 API나 구글 뉴스 RSS를 쓰더라도 결국 본문 파싱과 정제는 직접 해야 합니다.

Gemini의 Search Grounding을 쓰면 이 과정을 상당 부분 건너뛸 수 있습니다. "어느 언론사를 어떻게 파싱할까"가 아니라 **"어떤 키워드의 뉴스를 알고 싶은가"**만 정의하면, 검색·수집·요약·출처 인용까지 모델이 알아서 처리합니다.

이 글에서는 Search Grounding의 개념을 짚고, 이를 활용해 키워드 기반 뉴스 수집기를 만드는 과정을 정리합니다.

---

## Search Grounding이란

일반적인 LLM은 학습 시점(knowledge cutoff) 이후의 정보를 모릅니다. 그래서 "오늘 어떤 뉴스가 있었나?" 같은 질문에 답하지 못하거나, 그럴듯한 거짓(hallucination)을 만들어 냅니다. 뉴스처럼 시의성이 생명인 데이터에는 치명적입니다.

Search Grounding은 이 한계를 해결합니다. 모델이 답변을 생성하기 전에 **Google Search를 실제로 호출**해서 실시간 웹 결과를 가져오고, 그 내용을 근거로 답변을 작성합니다. 핵심은 다음과 같습니다.

- **실시간성** — 학습 시점과 무관하게 방금 올라온 기사까지 반영합니다.
- **출처 인용** — 답변에 사용된 기사의 URL과 인용 위치(`groundingMetadata`)를 함께 반환합니다. 뉴스 수집에서 출처는 필수입니다.
- **자동 판단** — 모델이 검색이 필요한지 스스로 판단하고, 필요하면 검색어를 자동 생성해 여러 번 검색합니다.

전체 흐름은 이렇습니다.

1. **프롬프트 전달** — 애플리케이션이 `google_search` 도구를 활성화한 상태로 질문을 보냅니다.
2. **검색 필요 판단** — 모델이 검색으로 답변이 개선될지 판단합니다.
3. **Google 검색 실행** — 필요하면 하나 이상의 검색어를 자동 생성해 검색합니다.
4. **결과 처리** — 검색 결과(기사들)를 종합해 답변을 구성합니다.
5. **근거가 있는 응답** — 답변 텍스트와 함께 검색어·웹 결과·인용 정보(`groundingMetadata`)를 반환합니다.

이 흐름을 보면 알 수 있습니다. **이건 사실상 "뉴스 검색 + 본문 파싱 + 요약"을 한 번의 API 호출로 처리하는 것**입니다.

---

## 왜 전통적인 뉴스 크롤러 대신 Search Grounding인가

전통적인 뉴스 크롤러와 Search Grounding 기반 수집기를 비교해 보겠습니다.

| 항목 | 전통적 뉴스 크롤러 | Search Grounding |
|------|-------------------|------------------|
| 언론사 선정 | 직접 지정 | 모델이 자동 검색 |
| 본문 파싱 | 언론사별 구현 필요 | 불필요 |
| JS 렌더링 | Headless 브라우저 필요 | 불필요 |
| 차단/봇 탐지 | 우회 로직 필요 | 해당 없음 |
| 기사 요약 | 별도 구현 (또는 LLM 추가 호출) | 한 번에 처리 |
| 출처 추적 | 직접 기록 | 자동 인용 |
| 비용 | 인프라/유지보수 | API 호출 비용 |

물론 만능은 아닙니다. **특정 언론사의 모든 기사를 빠짐없이 아카이빙하거나, 유료 구독 기사, 실시간 속보를 초 단위로 감지**해야 한다면 여전히 전통적 크롤러나 전용 뉴스 API가 맞습니다. 하지만 "특정 키워드의 주요 뉴스를 요약해서 모으는" 모니터링 용도라면 Search Grounding이 훨씬 효율적입니다.

---

## 준비: SDK 설치와 API 키

Google Gen AI SDK를 사용합니다. Gemini 2.0 이상부터 `google_search`가 도구(tool)로 제공됩니다.

```bash
pip install --upgrade google-genai
```

[Google AI Studio](https://aistudio.google.com/apikey)에서 API 키를 발급받아 환경 변수에 저장합니다.

**Linux/macOS:**

```bash
export GEMINI_API_KEY="your-api-key-here"
```

**Windows PowerShell:**

```powershell
$env:GEMINI_API_KEY="your-api-key-here"
```

---

## 기본 사용법

가장 단순한 형태입니다. `google_search` 도구만 추가하면 됩니다.

```python
from google import genai
from google.genai import types

client = genai.Client()

grounding_tool = types.Tool(
    google_search=types.GoogleSearch()
)

config = types.GenerateContentConfig(
    tools=[grounding_tool]
)

response = client.models.generate_content(
    model="gemini-2.5-flash",
    contents="최근 일주일간 '삼성전자 HBM' 관련 주요 뉴스를 정리해줘",
    config=config,
)

print(response.text)
```

`tools`에 `google_search`를 넣은 것만으로 모델이 알아서 뉴스를 검색하고, 그 결과를 바탕으로 답변합니다. 어떤 언론사를 긁을지 지정할 필요가 없습니다.

---

## 출처(기사 링크) 추출하기

뉴스 수집기라면 요약 텍스트뿐 아니라 **원문 기사 링크**가 반드시 필요합니다. 응답의 `grounding_metadata`에 이 정보가 들어 있습니다.

```python
def extract_sources(response):
    """응답에서 기사 제목과 URL을 추출"""
    sources = []
    candidate = response.candidates[0]

    metadata = getattr(candidate, "grounding_metadata", None)
    if not metadata or not metadata.grounding_chunks:
        return sources

    for chunk in metadata.grounding_chunks:
        if chunk.web:
            sources.append({
                "title": chunk.web.title,
                "uri": chunk.web.uri,
            })
    return sources


sources = extract_sources(response)
for i, src in enumerate(sources, 1):
    print(f"[{i}] {src['title']} - {src['uri']}")
```

- **`grounding_chunks`** — 답변의 근거가 된 기사 목록입니다. 각 청크의 `web.uri`, `web.title`로 출처를 얻습니다.
- 모델이 실제로 검색한 검색어는 `metadata.web_search_queries`에서 확인할 수 있습니다.

---

## 구조화된 뉴스 데이터로 수집하기

수집기라면 결과를 자유 텍스트가 아니라 **정형 데이터(JSON)**로 받는 것이 활용도가 높습니다. 다만 Search Grounding과 `response_schema`(JSON 모드)는 동시에 쓸 수 없는 경우가 있으므로, **프롬프트로 JSON 출력을 유도**하는 방식이 안전합니다.

```python
import json
import re
from google import genai
from google.genai import types

client = genai.Client()


def crawl_news(keyword: str, model: str = "gemini-2.5-flash") -> dict:
    """키워드에 대한 최신 뉴스를 검색해 구조화된 결과로 반환"""

    prompt = f"""'{keyword}' 키워드에 대해 Google 검색으로 최근 뉴스를 수집하고,
아래 JSON 형식으로만 응답해줘. 코드블록이나 설명은 붙이지 마.
최신순으로 최대 5건만 추려줘.

[
  {{
    "headline": "기사 제목",
    "summary": "기사 핵심 내용 2~3문장 요약",
    "press": "언론사명 (알 수 있으면, 없으면 빈 문자열)",
    "date": "보도 날짜 (알 수 있으면, 없으면 빈 문자열)"
  }}
]
"""

    config = types.GenerateContentConfig(
        tools=[types.Tool(google_search=types.GoogleSearch())],
        temperature=0.2,
    )

    response = client.models.generate_content(
        model=model,
        contents=prompt,
        config=config,
    )

    articles = parse_json(response.text)
    sources = extract_sources(response)

    return {
        "keyword": keyword,
        "articles": articles,
        "sources": sources,
    }


def parse_json(text: str) -> list:
    """모델 응답에서 JSON 배열만 추출해 파싱"""
    match = re.search(r"\[.*\]", text, re.DOTALL)
    if not match:
        return []
    try:
        return json.loads(match.group(0))
    except json.JSONDecodeError:
        return []


if __name__ == "__main__":
    result = crawl_news("생성형 AI 규제")

    print(f"# '{result['keyword']}' 뉴스\n")
    for art in result["articles"]:
        print(f"## {art['headline']}")
        print(f"- {art.get('press', '')} | {art.get('date', '')}")
        print(art["summary"], "\n")

    print("## 출처")
    for i, src in enumerate(result["sources"], 1):
        print(f"[{i}] {src['title']} - {src['uri']}")
```

`temperature=0.2`로 낮춰 출력 형식의 일관성을 높였습니다. `parse_json`은 모델이 코드블록 등을 덧붙이는 경우를 대비해 JSON 배열만 정규식으로 추출합니다.

---

## 여러 키워드를 매일 자동으로 수집하기

수집기다운 형태로 확장해 보겠습니다. 모니터링할 키워드들을 돌면서 결과를 날짜별 파일로 저장합니다.

```python
import time
import json
from datetime import datetime
from pathlib import Path

KEYWORDS = [
    "생성형 AI 규제",
    "반도체 수출 규제",
    "전기차 배터리 신기술",
]


def run_news_crawler(keywords: list, output_dir: str = "news_results"):
    Path(output_dir).mkdir(exist_ok=True)
    today = datetime.now().strftime("%Y%m%d")

    all_results = []
    for keyword in keywords:
        print(f"뉴스 수집 중: {keyword}")
        try:
            result = crawl_news(keyword)
            all_results.append(result)
        except Exception as e:
            print(f"  실패: {e}")
        time.sleep(2)  # 요청 간 간격

    output_path = Path(output_dir) / f"{today}.json"
    output_path.write_text(
        json.dumps(all_results, ensure_ascii=False, indent=2),
        encoding="utf-8",
    )
    print(f"저장 완료: {output_path}")


if __name__ == "__main__":
    run_news_crawler(KEYWORDS)
```

이 스크립트를 cron(Linux) 또는 작업 스케줄러(Windows)에 등록하면, 매일 아침 관심 키워드의 주요 뉴스를 자동으로 모으는 셈입니다. 별도의 크롤링 인프라 없이 API 호출만으로 동작합니다. 결과를 Slack 웹훅이나 이메일로 보내면 간단한 **뉴스 브리핑 봇**이 됩니다.

---

## 실전 팁

실제로 뉴스 모니터링을 운영하면서 도움이 되는 몇 가지입니다.

- **기간 한정** — 프롬프트에 "최근 24시간", "이번 주" 같은 기간을 명시하면 오래된 기사가 섞이는 것을 줄일 수 있습니다. 매일 돌리는 봇이라면 "지난 하루"로 좁히세요.
- **도메인 필터링** — `GoogleSearch(exclude_domains=["example.com"])`로 특정 도메인을 결과에서 제외할 수 있습니다. 광고성·낚시성 사이트를 거르거나, 반대로 신뢰하는 언론사 위주로 유도하는 데 활용합니다.
- **중복 제거** — 같은 사건을 여러 언론사가 보도하면 비슷한 기사가 중복됩니다. 헤드라인 유사도로 후처리하거나, 프롬프트에 "유사 기사는 하나로 묶어줘"를 추가하세요.
- **모델 선택** — 대량 수집에는 비용·속도가 유리한 `gemini-2.5-flash`나 `flash-lite`가 적합합니다. 기사 간 맥락 분석이 필요하면 `pro` 계열을 씁니다.
- **검색어 확인** — `web_search_queries`를 로그로 남기면 모델이 어떤 검색을 했는지 디버깅할 수 있습니다. 원하는 기사가 안 나올 때 키워드나 프롬프트를 조정하는 단서가 됩니다.
- **Search Suggestions 표시 의무** — Google의 정책상, Grounding 결과를 프로덕션 서비스에 노출할 때는 응답에 포함된 **Search Suggestions를 함께 표시**해야 합니다. 외부로 배포하는 뉴스 서비스라면 반드시 확인하세요.
- **비용 관리** — Grounding은 일반 호출보다 비용이 높습니다. 키워드 수와 수집 주기를 적절히 조절하세요.

---

## 주의할 점

편리한 만큼, 뉴스를 다룬다는 점에서 반드시 짚고 넘어가야 할 한계와 위험이 있습니다.

### 1. 저작권 문제

뉴스 기사 본문은 언론사의 저작물입니다. Search Grounding으로 얻은 결과를 다룰 때는 다음을 유의해야 합니다.

- **요약은 되지만 전문 복제는 안 됩니다.** 모델이 생성하는 것은 기사에 대한 요약이지만, 요약이라도 원문을 그대로 옮기거나 실질적으로 대체할 정도라면 저작권·부정경쟁 이슈가 생길 수 있습니다. 헤드라인 자체도 보호 대상이 될 수 있습니다.
- **출처를 반드시 명시하고 원문으로 연결하세요.** `grounding_chunks`로 얻은 기사 링크를 함께 노출해, 사용자가 원문을 직접 보도록 유도하는 것이 안전합니다.
- **재배포·상업적 활용은 별도 검토가 필요합니다.** 수집한 요약을 외부에 서비스하거나 상업적으로 쓰려면, 해당 언론사의 이용 약관이나 뉴스 저작권 신탁(예: 한국언론진흥재단) 등을 확인해야 합니다. 내부 모니터링 용도와 외부 서비스는 법적 무게가 다릅니다.

### 2. 구글의 약관(Terms of Service)

Search Grounding은 Google의 API를 통해 제공되며, 사용에는 약관 준수가 전제됩니다.

- **Search Suggestions 표시 의무.** Grounding 응답을 사용자에게 노출하는 서비스라면, 응답에 포함된 Search Suggestions를 **그대로 표시해야 합니다.** 이를 제거하거나 가공하면 약관 위반입니다.
- **검색 결과의 별도 저장·재사용 제한.** Grounding이 반환한 검색 결과나 메타데이터를 Google이 의도하지 않은 방식으로 대량 축적·재배포하는 것은 약관에 저촉될 수 있습니다. "응답을 캐싱해 우리 검색 DB를 만든다" 같은 용도는 위험합니다.
- **무료 티어의 데이터 활용 정책.** 무료(AI Studio) 키와 유료(결제 연동) 사용의 데이터 처리 정책이 다릅니다. 민감한 키워드나 내부 정보를 프롬프트에 넣는다면 어떤 정책이 적용되는지 확인하세요.
- 약관은 수시로 바뀌므로, 운영 전 [Gemini API 추가 약관](https://ai.google.dev/gemini-api/terms)을 직접 확인해 볼 필요가 있습니다.

### 3. 최신성·정확성이 보장되지 않음

Search Grounding은 "실시간 검색"을 하지만, 그것이 **완전성과 최신성을 보장하지는 않습니다.**

- **속보를 놓칠 수 있습니다.** 검색 색인에 반영되기까지 시차가 있어, 방금 나온 속보는 누락될 수 있습니다. 초 단위·분 단위 실시간 감지가 필요하면 전용 뉴스 API가 맞습니다.
- **전수 수집이 아닙니다.** 모델은 검색 결과 중 일부만 골라 답변합니다. "해당 키워드의 모든 기사"를 모으는 것이 아니라 "대표적인 일부"를 모으는 것입니다.
- **요약 과정에서 왜곡·환각 가능성.** 출처가 있어도 모델이 기사 내용을 잘못 요약하거나, 여러 기사를 뒤섞어 사실과 다른 문장을 만들 수 있습니다. **요약을 그대로 신뢰하지 말고 중요한 사안은 원문으로 교차 확인**해야 합니다.
- **날짜·언론사 정보의 부정확성.** 프롬프트로 받은 `date`, `press` 값은 모델 추정치일 수 있습니다. 정확한 보도 시점이 중요하다면 링크의 원문에서 확인하세요.

요약하면, 이 방식은 **"빠르고 가벼운 뉴스 모니터링"에는 탁월하지만, "정확성·완전성·실시간성이 법적·업무적으로 중요한 용도"에는 보조 수단으로만 써야 합니다.** 최종 판단은 항상 원문 기사를 근거로 하세요.

---

## 마무리

Search Grounding 기반 뉴스 수집기의 핵심은 **"수집 로직을 작성하지 않는다"**는 점입니다. 언론사별 HTML 파서도, 헤드리스 브라우저도, 차단 우회 로직도 필요 없습니다. 어떤 키워드의 뉴스를 알고 싶은지 프롬프트로 정의하면, 검색부터 본문 요약·출처 인용까지 모델이 처리합니다.

물론 모든 뉴스 크롤링을 대체하지는 못합니다. 특정 언론사 전수 아카이빙, 유료 구독 기사, 초 단위 속보 감지에는 전용 뉴스 API나 전통적 방식이 적합합니다. 하지만 **"관심 키워드의 주요 뉴스를 요약해서 매일 받아보는"** 모니터링 용도라면, Search Grounding은 가장 적은 코드로 가장 빠르게 결과를 얻을 수 있습니다.

키워드 목록과 스케줄러만 있으면, 나만의 뉴스 브리핑 봇을 오늘 바로 만들 수 있습니다.

**참고 자료:**

- [Grounding with Google Search - Gemini API 공식 문서](https://ai.google.dev/gemini-api/docs/google-search)
- [Grounding with Google Search - Vertex AI 문서](https://cloud.google.com/vertex-ai/generative-ai/docs/grounding-with-search)
- [Google Gen AI SDK (Python)](https://googleapis.github.io/python-genai/)
- [Google AI Studio - API 키 발급](https://aistudio.google.com/apikey)
