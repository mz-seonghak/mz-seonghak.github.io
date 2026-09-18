---
layout: post
title: "LLM 토큰 사용량 미터링 유틸리티 3가지 — Tokscale, CodeBurn, Tokuin"
date: 2026-06-10
tags: [llm, token, metering, monitoring, tools]
---

AI 코딩 도구를 많이 쓰면서 이제 토큰 관리도 중요하게 되었습니다.

Claude Code, Codex, Cursor, OpenClaw 같은 AI 코딩 에이전트를 매일 쓰는 시대입니다. 정액제 플랜을 쓰든 API 종량제를 쓰든, 토큰 사용량을 파악하지 못하면 어디서 비용이 새는지 알 수 없습니다. 어떤 모델이 토큰을 많이 먹는지, 어떤 프로젝트가 비용을 많이 쓰는지, 하루에 얼마씩 태우고 있는지를 알아야 절약 방법도 찾을 수 있습니다.

다행히 이런 요구에 맞는 토큰 미터링(metering) 도구들이 여럿 나와 있습니다. 공통점은 별도의 프록시나 래퍼 설치 없이, **AI 코딩 도구가 로컬 디스크에 이미 남기고 있는 세션 로그를 읽어서** 소프트웨어별, 모델별, 날짜별 사용량과 비용을 보여준다는 것입니다. 이 글에서는 많이 쓰이는 3가지 도구를 소개합니다.

| 도구 | 저장소 | 언어 | 특징 한 줄 요약 |
|------|--------|------|----------------|
| Tokscale | [junhoyeo/tokscale](https://github.com/junhoyeo/tokscale) | Rust + TypeScript | 가장 폭넓은 플랫폼 지원 + 글로벌 리더보드 |
| CodeBurn | [getagentseal/codeburn](https://github.com/getagentseal/codeburn) | TypeScript | 비용 옵저버빌리티에 특화된 TUI 대시보드 |
| Tokuin | [nooscraft/tokuin](https://github.com/nooscraft/tokuin) | Rust | 토큰 추정 + 비용 분석 + 프롬프트 압축 |

---

## Tokscale — 한국 개발자가 만든 토큰 추적기

[Tokscale](https://github.com/junhoyeo/tokscale)은 제작자([junhoyeo](https://github.com/junhoyeo))가 한국인이라 한국 개발자들 사이에서 특히 많이 쓰이는 도구입니다. 물론 한국에서만 쓰는 도구는 아니고, GitHub 스타가 3,500개를 넘을 정도로 글로벌하게도 인기가 있습니다.

핵심 특징은 다음과 같습니다.

- **압도적인 플랫폼 지원** — Claude Code, Codex CLI, Cursor, Gemini CLI, OpenCode, OpenClaw, Copilot CLI, Amp, Droid, Kimi CLI, Qwen CLI, Roo Code, Cline, Zed, Goose 등 사실상 시중의 AI 코딩 도구 대부분을 커버합니다.
- **네이티브 Rust 코어** — 로그 파싱과 집계를 Rust로 처리해서 세션 데이터가 많이 쌓여도 빠릅니다.
- **상세한 토큰 분류** — 입력/출력뿐 아니라 캐시 읽기/쓰기, 추론(reasoning) 토큰까지 구분해서 보여줍니다.
- **실시간 가격 정보** — LiteLLM 가격 데이터를 가져와서 최신 모델 단가로 비용을 계산합니다.
- **글로벌 리더보드** — 사용량을 [tokscale.ai](https://tokscale.ai)에 제출하면 GitHub 잔디 스타일의 2D/3D 기여 그래프와 함께 전 세계 순위에 올라갑니다. "내가 토큰을 이만큼 태웠다"를 자랑(?)하는 문화가 형성되어 있습니다.

설치 없이 바로 실행할 수 있습니다.

```bash
# npx로 바로 실행
npx tokscale@latest

# 인터랙티브 TUI 실행 (기본)
tokscale

# 모델별/월별 집계를 JSON으로 출력
tokscale models --json
tokscale monthly --json

# 특정 클라이언트만 필터링
tokscale --client claude,cursor
```

리더보드에 참여하려면 GitHub 계정으로 로그인하고 제출하면 됩니다.

```bash
tokscale login
tokscale submit
```

Spotify Wrapped처럼 연말 회고 이미지를 만들어 주는 `tokscale wrapped` 같은 재미 요소도 있습니다. 사용량 추적을 단순한 회계가 아니라 일종의 놀이로 만든 점이 인기 비결인 것 같습니다.

---

## CodeBurn — 비용이 어디로 새는지 보여주는 대시보드

[CodeBurn](https://github.com/getagentseal/codeburn)은 해외 개발자들이 많이 쓰는 도구입니다. 공개 첫 주에 GitHub 스타 3,400개를 넘겼을 정도로 빠르게 퍼졌고, 현재는 7,000개를 넘었습니다.

Tokscale이 "얼마나 썼는가"에 집중한다면, CodeBurn은 한 걸음 더 들어가서 **"어디서, 왜 새고 있는가"**를 보여주는 옵저버빌리티 도구에 가깝습니다.

- **TUI 대시보드** — 터미널 안에서 실행되는 대시보드로, 30초마다 자동 갱신됩니다. 계정 생성도, 로그인도, 외부로 나가는 텔레메트리도 없습니다. 로컬 로그만 읽습니다.
- **다차원 분석** — 작업 유형(task type)별, 모델별, 도구별, 프로젝트별, 날짜별로 비용을 쪼개서 보여줍니다. "이번 주에 어떤 리포지토리가 토큰을 제일 많이 태웠나" 같은 질문에 바로 답할 수 있습니다.
- **`optimize` 명령** — 토큰 낭비 패턴을 찾아내고 복사해서 바로 쓸 수 있는 개선안을 제시합니다.
- **`yield` 명령** — AI에 쓴 비용과 git 커밋 결과를 연결해서, 생산적인 작업과 되돌려진(reverted) 작업을 구분합니다. "토큰을 태워서 실제로 남은 게 뭔가"를 측정하는 독특한 기능입니다.
- **정액제 사용자에게도 유용** — Claude Max 같은 정액 플랜을 쓰더라도 종량제 환산 금액을 보여주기 때문에, 내 플랜이 본전을 뽑고 있는지 직관적으로 알 수 있습니다.

설치와 기본 사용법입니다.

```bash
npm install -g codeburn

codeburn                  # 인터랙티브 대시보드 (기본 7일)
codeburn today            # 오늘 사용량
codeburn month            # 이번 달 사용량
codeburn status           # 한 줄 요약 (오늘 + 이번 달)
codeburn models           # 모델별 토큰/비용 테이블
codeburn optimize         # 낭비 패턴 진단
codeburn report --from 2026-06-01 --to 2026-06-10   # 날짜 범위 지정
```

`--provider` 플래그로 특정 도구만 골라 볼 수도 있습니다.

```bash
codeburn report --provider claude
codeburn today --provider codex
codeburn export --provider cursor
```

macOS에서는 메뉴 바 앱도 제공해서 터미널을 열지 않고도 실시간 사용량을 확인할 수 있습니다.

---

## Tokuin — 쓰기 전에 재고, 줄이는 도구

[Tokuin](https://github.com/nooscraft/tokuin)은 앞의 두 도구와 결이 조금 다릅니다. Tokscale과 CodeBurn이 **이미 쓴** 토큰을 사후에 집계하는 도구라면, Tokuin은 **쓰기 전에** 토큰을 추정하고, 비용을 예측하고, 나아가 프롬프트 자체를 압축해서 토큰을 줄이는 데 초점이 있습니다.

- **토큰 추정** — 텍스트, 채팅 트랜스크립트, JSON 페이로드를 넣으면 system/user/assistant 역할별로 토큰을 계산해 줍니다.
- **멀티 모델 비교** — `--compare` 옵션으로 OpenAI, Anthropic, OpenRouter 모델 간 토큰 수와 비용 차이를 한눈에 비교합니다.
- **비용 예측** — `--estimate-cost`로 프롬프트당, 실행당 예상 비용을 미리 계산합니다.
- **프롬프트 압축** — v0.2.0에서 추가된 Hieratic 압축 엔진이 핵심 기능입니다. 프롬프트를 30~90%까지 압축하면서 의미 유사도, 핵심 지시문 보존 여부, 구조 무결성 같은 품질 지표로 검증할 수 있습니다.
- **로드 테스트** — 동시성 제어, 재시도 로직을 갖춘 API 부하 테스트로 지연 시간과 비용 리포트를 뽑아줍니다.

```bash
# 설치 (Linux/macOS)
curl -fsSL https://raw.githubusercontent.com/nooscraft/tokuin/main/install.sh | bash

# 프롬프트 압축 (light: 30~50%, medium: 50~70%, aggressive: 70~90%)
tokuin compress prompt.txt --level medium

# 압축 품질 검증
tokuin compress prompt.txt --quality

# 모델 간 토큰/비용 비교
tokuin --model gpt-4 --compare
```

저는 Windows 환경이라 아직 직접 써보지 못했습니다. 릴리스에 Windows용 바이너리(`x86_64-pc-windows-msvc`)가 올라오기 시작했으니 곧 제대로 써볼 수 있을 것 같습니다. 써본 사람들 평은 괜찮은 편입니다. 특히 토큰 비용의 대부분이 입력 토큰에서 나오는 요즘 구조에서, 프롬프트 압축으로 입력을 줄인다는 접근은 사후 집계와는 다른 차원의 절약 방법입니다.

---

## 어떤 걸 쓰면 좋을까

세 도구의 용도가 조금씩 다르므로 목적에 맞게 고르면 됩니다.

| 목적 | 추천 도구 |
|------|----------|
| 여러 AI 코딩 도구의 사용량을 한곳에서 보고 싶다 | Tokscale |
| 리더보드, 잔디 그래프 같은 재미 요소를 원한다 | Tokscale |
| 비용이 새는 지점을 찾아 최적화하고 싶다 | CodeBurn |
| 정액 플랜의 본전 여부를 확인하고 싶다 | CodeBurn |
| 프롬프트를 보내기 전에 토큰/비용을 예측하고 싶다 | Tokuin |
| 프롬프트 압축으로 입력 토큰 자체를 줄이고 싶다 | Tokuin |

사실 Tokscale과 CodeBurn은 같은 로컬 로그를 읽기 때문에 둘 다 설치해 두고 비교해 봐도 부담이 없습니다. 계정이나 프록시 설정 없이 명령어 하나로 바로 결과가 나오니, 일단 `npx tokscale@latest`나 `npx codeburn` 한 번 실행해 보는 것부터 시작하면 됩니다.

토큰은 곧 돈입니다. AI 도구를 매일 쓰는 시대에 토큰 미터링은 선택이 아니라 기본 위생에 가깝습니다. 측정하지 않으면 절약할 수 없습니다.
