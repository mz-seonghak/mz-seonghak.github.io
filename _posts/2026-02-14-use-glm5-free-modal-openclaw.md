---
layout: post
title: "GLM-5를 무료로 사용하는 방법: Modal API + OpenClaw 설정 가이드"
date: 2026-02-14
tags: [glm-5, openclaw, modal, llm, api]
---

2026년 2월 11일, Z.ai에서 GLM-5를 공개했습니다. 745B 파라미터의 오픈소스 LLM으로, MIT 라이선스로 배포되어 상업적 활용까지 가능합니다. 더 좋은 소식은 Modal 플랫폼이 Z.ai와의 파트너십을 통해 2026년 4월 30일까지 GLM-5를 완전 무료로 제공하고 있다는 것입니다.

이 글에서는 Modal에서 API 토큰을 발급받고, OpenClaw에서 GLM-5를 사용할 수 있도록 설정하는 전체 과정을 정리합니다.

## GLM-5는 어떤 모델인가

GLM-5의 핵심 특징은 다음과 같습니다.

| 항목 | 내용 |
|------|------|
| 파라미터 | 745B (7,450억 개) |
| 아키텍처 | Mixture-of-Experts (MoE) |
| 양자화 | FP8 |
| 컨텍스트 윈도우 | 192,000 토큰 |
| 라이선스 | MIT |
| 강점 | 장기 호라이존 작업, 시스템 엔지니어링, 코드 생성 |

MoE 아키텍처는 모든 파라미터를 항상 사용하는 것이 아니라, 작업에 따라 필요한 전문가(Expert)만 활성화하는 방식입니다. 코딩 작업에는 코딩 전문가를, 창작 작업에는 창작 전문가를 사용하여 효율적으로 추론합니다.

벤치마크에서는 Claude Opus 4.6이나 GPT-5.3 Codex 같은 최첨단 상용 모델과 대등한 결과를 보여주었으며, 특히 코드 생성, 복잡한 문제 해결, 장기 맥락 이해에서 뛰어난 성능을 보이고 있습니다.

다만 FP8 양자화 상태에서도 약 700GB의 저장 공간이 필요하기 때문에 로컬에서 직접 실행하는 것은 사실상 불가능합니다. 그래서 Modal 같은 클라우드 플랫폼을 통해 API로 사용하는 것이 현실적인 방법입니다.

## Modal에서 GLM-5 무료 사용하기

[Modal](https://modal.com)은 AI/ML 모델을 쉽게 배포하고 실행할 수 있는 클라우드 플랫폼입니다. GLM-5 공개 전에 Z.ai와 파트너십을 맺고 모델 최적화 작업을 진행했기 때문에, 일반 사용자도 간편하게 GLM-5를 사용할 수 있습니다.

**무료 제공 조건은 다음과 같습니다.**

- 무료 기간: 2026년 4월 30일까지
- API 형식: OpenAI 호환 (chat/completions)
- 동시 요청: 1개 (개인 사용 위주)
- 토큰 제한: 없음
- 생성 속도: 초당 30~75 토큰

동시 요청이 1개로 제한되어 있으므로 프로덕션 환경에는 적합하지 않지만, 개인 개발이나 학습 목적으로는 충분합니다.

### 1단계: Modal 계정 생성

[Modal 웹사이트](https://modal.com)에 접속하여 계정을 생성합니다. GitHub 계정으로 간편 로그인이 가능합니다.

### 2단계: GLM-5 API 토큰 발급

[GLM-5 토큰 발급 페이지](https://modal.com/glm-5-endpoint)에서 API 토큰을 생성합니다.

토큰 생성 후 표시되는 정보는 다음과 같습니다.

| 항목 | 값 |
|------|-----|
| Base URL | `https://api.us-west-2.modal.direct/v1` |
| Model ID | `zai-org/GLM-5-FP8` |
| API 형식 | OpenAI 호환 (chat/completions) |

**주의:** API 토큰은 발급 후 다시 확인할 수 없습니다. 반드시 안전한 곳에 저장해 두세요.

### 3단계: 환경 변수에 API 토큰 저장

API 토큰을 환경 변수로 설정합니다. 설정 파일에 토큰을 직접 하드코딩하는 것은 보안상 권장되지 않습니다.

**Linux/macOS** (`~/.bashrc` 또는 `~/.zshrc`에 추가):

```bash
export LLM_BACKEND_API_KEY="modal_research_xxxxx...your-token-here"
```

설정 후 `source ~/.bashrc` 또는 터미널을 재시작하면 적용됩니다.

**Windows PowerShell:**

```powershell
$env:LLM_BACKEND_API_KEY="modal_research_xxxxx...your-token-here"
```

영구적으로 설정하려면 시스템 환경 변수에 등록하거나 PowerShell 프로필에 추가하세요.

### 토큰 동작 확인

curl로 간단히 테스트할 수 있습니다.

```bash
curl https://api.us-west-2.modal.direct/v1/chat/completions \
  -H "Authorization: Bearer $LLM_BACKEND_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "zai-org/GLM-5-FP8",
    "messages": [{"role": "user", "content": "Hello, GLM-5!"}],
    "max_tokens": 100
  }'
```

응답이 정상적으로 오면 토큰이 올바르게 설정된 것입니다.

## OpenClaw에서 GLM-5 설정하기

[이전 글](20260208_2254%20-%20GCP에%20Openclaw%20설치하기.md)에서 GCP에 OpenClaw를 설치하는 방법을 다뤘습니다. 이미 OpenClaw가 설치되어 있다면 설정 파일만 수정하면 GLM-5를 바로 사용할 수 있습니다.

### OpenClaw 설정 파일 위치

```
# Linux/macOS
~/.openclaw/openclaw.json

# Windows
C:\Users\{username}\.openclaw\openclaw.json
```

### GLM-5 공급자 추가

`openclaw.json` 파일의 `models` 섹션에 Modal 공급자를 추가합니다.

```json
{
  "models": {
    "mode": "merge",
    "providers": {
      "modal": {
        "baseUrl": "https://api.us-west-2.modal.direct/v1",
        "apiKey": "${LLM_BACKEND_API_KEY}",
        "api": "openai-completions",
        "models": [
          {
            "id": "zai-org/GLM-5-FP8",
            "name": "GLM-5",
            "reasoning": true,
            "input": ["text"],
            "cost": {
              "input": 0,
              "output": 0,
              "cacheRead": 0,
              "cacheWrite": 0
            },
            "contextWindow": 192000,
            "maxTokens": 8192
          }
        ]
      }
    }
  }
}
```

각 항목의 의미는 다음과 같습니다.

- **`mode: "merge"`** — 기존 공급자를 유지하면서 새 공급자를 추가합니다. 기존에 사용하던 Claude나 GPT 설정이 있다면 그대로 유지됩니다.
- **`apiKey: "${LLM_BACKEND_API_KEY}"`** — 환경 변수에서 토큰을 읽어옵니다. 설정 파일에 토큰을 직접 넣지 않아도 됩니다.
- **`api: "openai-completions"`** — Modal의 GLM-5 엔드포인트가 OpenAI API 호환이므로 이 형식을 사용합니다.
- **`reasoning: true`** — GLM-5의 추론 능력을 활성화합니다.
- **`cost`** — 모든 값이 0입니다. 무료 기간이므로 비용이 발생하지 않습니다.
- **`contextWindow: 192000`** — 192K 토큰의 긴 컨텍스트를 지원합니다.
- **`maxTokens: 8192`** — 한 번의 응답에서 생성할 수 있는 최대 토큰 수입니다.

### 설정 검증

설정을 저장한 후 OpenClaw를 재시작하고 상태를 확인합니다.

```bash
openclaw status
openclaw models list
```

모델 목록에 `GLM-5`가 표시되면 설정이 완료된 것입니다. OpenClaw 채팅 인터페이스에서 GLM-5 모델을 선택하여 사용할 수 있습니다.

## 다른 모델과 비교

OpenClaw에서 여러 모델을 전환하며 사용할 수 있으니, 작업 유형에 따라 적합한 모델을 선택하면 됩니다.

| 항목 | GLM-5 | Claude Opus 4.6 | GPT-5.3 Codex | DeepSeek-V3 |
|------|-------|-----------------|---------------|-------------|
| 파라미터 | 745B | ~400B (추정) | ~500B+ (추정) | 685B |
| 라이선스 | MIT (오픈소스) | 상용 | 상용 | MIT (오픈소스) |
| 강점 | 장기 작업, 시스템 엔지니어링 | 일반 대화, 분석 | 코드 생성 | 코딩, 수학, 추론 |
| 비용 | 무료 (4/30까지) | 유료 | 유료 | 무료 API 제공 |
| 컨텍스트 | 192K | 200K | 128K | 128K |

장기 프로젝트에는 GLM-5의 긴 컨텍스트 윈도우와 장기 맥락 유지 능력이 유리합니다. 빠른 코드 생성에는 GPT-5.3 Codex나 DeepSeek-V3가, 일반적인 대화와 분석에는 Claude Opus 4.6이 적합합니다. 예산이 제한적이라면 GLM-5와 DeepSeek-V3의 무료 옵션을 먼저 테스트해 보세요.

## 마무리

GLM-5는 오픈소스 모델 중에서 코딩과 에이전트 작업에 최고 수준의 성능을 보여주고 있습니다. Modal이 4월 말까지 무료로 제공하고 있으니, 이 기회에 OpenClaw와 함께 사용해 보는 것을 추천합니다. OpenAI API 호환 엔드포인트를 제공하기 때문에 OpenClaw 외에도 OpenAI API를 지원하는 대부분의 도구에서 바로 사용할 수 있습니다.

다만 동시 요청이 1개로 제한되어 있으므로, 본격적인 프로덕션 환경에서는 Modal의 유료 플랜이나 직접 호스팅을 검토해야 합니다. 개인 에이전트 용도로 24시간 돌리는 수준이라면 현재 무료 티어로 충분합니다.

**참고 자료:**

- [Modal - Try GLM-5 on Modal](https://modal.com/blog/try-glm-5)
- [GLM-5 토큰 발급 페이지](https://modal.com/glm-5-endpoint)
- [OpenClaw 공식 문서](https://github.com/openclaw/openclaw)
- [GLM-5 무료 사용법 원문 (제임스의 AI 실전 노트)](https://fornewchallenge.tistory.com/entry/%F0%9F%86%93-GLM-5-%EB%AC%B4%EB%A3%8C-%EC%82%AC%EC%9A%A9%EB%B2%95-Modal-API%EB%B6%80%ED%84%B0-OpenClaw-%EC%84%A4%EC%A0%95%EA%B9%8C%EC%A7%80-%EC%99%84%EB%B2%BD-%EA%B0%80%EC%9D%B4%EB%93%9C)
