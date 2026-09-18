---
layout: post
title: "GCP에 OpenClaw 설치하기"
date: 2026-02-08
tags: [openclaw, gcp, compute-engine, docker, ai-agent]
---

OpenClaw를 사용하기 위해 Mac Mini를 구입하는 케이스가 미국에서 늘고 있다는 뉴스에 한국에서도 너도나도 맥미니를 사겠다는 사람들이 늘고 있습니다.

사람들이 맥미니를 선택하는 이유는 이렇습니다. 알려진 브랜드 중에서 안정적인 편이고, 쿨링팬 소리도 작고, macOS가 유닉스 기반이라 개발 환경과의 호환성이 좋습니다. 스펙이 좋다면 로컬 LLM을 올려서 사용할 수도 있고, 무엇보다 작아서 구석에 두고 24시간 에이전트를 돌리기에 좋다는 것이 가장 큰 이유입니다.

## 맥미니가 최선의 선택은 아닙니다

하지만 맥미니가 최선의 선택인지는 다시 생각해볼 필요가 있습니다.

**가격 대비 성능 문제.** 동일한 가격대에 훨씬 좋은 스펙의 미니PC가 많습니다. 레노버 ThinkCentre, Intel NUC, Beelink 같은 브랜드들은 비슷하거나 더 낮은 가격에 충분한 성능을 제공합니다.

**로컬 LLM의 한계.** 맥미니에 로컬 LLM을 설치해서 OpenClaw를 돌리겠다는 분들이 있는데, 현실적으로 로컬에서 돌릴 수 있는 경량 모델은 자율형 에이전트가 요구하는 수준의 판단력을 제공하지 못합니다. OpenClaw가 똑똑하게 일하려면 Anthropic의 Claude나 OpenAI의 GPT처럼 고성능 모델을 API로 사용하는 것이 사실상 필수입니다.

**보안 문제는 여전합니다.** OpenClaw는 파일 시스템, 브라우저, 메시징 플랫폼까지 광범위한 접근 권한을 갖고 24시간 동작합니다. 맥미니에 직접 설치하더라도 프롬프트 인젝션이나 악성 스킬을 통한 공격 위험이 존재합니다. 보안을 강화하려면 Docker 컨테이너 안에서 샌드박스로 실행하거나, 아예 클라우드 VM에 격리해서 운영하는 것이 바람직합니다.

## 클라우드 VM이 더 나은 이유

맥미니를 사지 않고도 24시간 에이전트를 운영할 수 있는 방법이 있습니다. 바로 클라우드 VM에 설치하는 것입니다.

- **비용이 저렴합니다.** GCP의 e2-small 인스턴스 기준 월 약 12달러(약 17,000원)면 충분합니다. 맥미니 구입비(최소 80만 원 이상) + 전기세와 비교하면 압도적으로 경제적입니다.
- **24시간 안정적으로 운영됩니다.** 구글 데이터센터의 인프라에서 동작하므로 정전, 네트워크 불안정, 하드웨어 고장 걱정이 없습니다.
- **보안 격리가 쉽습니다.** VM 자체가 개인 환경과 분리되어 있어 OpenClaw가 로컬 파일이나 홈 네트워크에 접근할 수 없습니다.
- **확장과 관리가 편합니다.** 메모리가 부족하면 머신 타입만 바꾸면 되고, 문제가 생기면 VM을 재생성하면 됩니다.

물론 단점도 있습니다. 집안의 IoT 디바이스를 제어하거나 웹캠을 사용하는 등의 작업은 클라우드 VM에서 직접 수행할 수 없습니다. 하지만 앞서 말했듯이 OpenClaw가 홈 디바이스까지 제어하게 하려면 보안을 매우 신경 써야 합니다. 자료 조사, 리포트 작성, 번역, 애플리케이션 개발, 스케줄 관리 등 대부분의 작업은 홈 디바이스를 제어할 필요가 없기 때문에 VM에 설치하는 것이 가장 합리적입니다.

## GCP에 OpenClaw 설치하기

이제 실제로 GCP Compute Engine에 Docker를 사용해서 OpenClaw를 설치하는 방법을 단계별로 설명하겠습니다.

### 사전 준비

- GCP 계정 (신규 가입 시 $300 크레딧 90일 무료)
- gcloud CLI 또는 GCP 콘솔 접근
- LLM API 키 (Anthropic 또는 OpenAI)
- 약 20~30분의 시간

### 1단계: GCP 프로젝트 생성

gcloud CLI를 설치한 뒤 프로젝트를 생성합니다.

```bash
gcloud init
gcloud auth login

# 프로젝트 생성
gcloud projects create my-openclaw-project --name="OpenClaw Gateway"
gcloud config set project my-openclaw-project

# Compute Engine API 활성화
gcloud services enable compute.googleapis.com
```

GCP 콘솔을 통해 생성할 수도 있습니다. IAM & Admin > Create Project에서 프로젝트를 만들고, 결제를 활성화한 뒤 Compute Engine API를 켜면 됩니다.

### 2단계: VM 인스턴스 생성

OpenClaw에 적합한 머신 타입은 다음과 같습니다.

| 타입 | 스펙 | 월 비용 | 비고 |
|------|------|---------|------|
| e2-micro | 2 vCPU(공유), 1GB RAM | ~$6/월 | 프리 티어 대상. 부하 시 OOM 가능 |
| e2-small | 2 vCPU, 2GB RAM | ~$12/월 | **권장**. 안정적 운영 가능 |

```bash
gcloud compute instances create openclaw-gateway \
  --zone=us-central1-a \
  --machine-type=e2-small \
  --boot-disk-size=20GB \
  --image-family=debian-12 \
  --image-project=debian-cloud
```

비용을 최소화하고 싶다면 e2-micro로 시작해서 OOM이 발생할 경우 e2-small로 올리는 전략도 괜찮습니다.

### 3단계: VM에 SSH 접속 및 Docker 설치

```bash
# VM에 SSH 접속
gcloud compute ssh openclaw-gateway --zone=us-central1-a

# Docker 설치
sudo apt-get update
sudo apt-get install -y git curl ca-certificates
curl -fsSL https://get.docker.com | sudo sh
sudo usermod -aG docker $USER

# 로그아웃 후 재접속 (docker 그룹 적용)
exit
gcloud compute ssh openclaw-gateway --zone=us-central1-a

# 설치 확인
docker --version
docker compose version
```

### 4단계: OpenClaw 클론 및 디렉토리 구성

```bash
# 소스 클론
git clone https://github.com/openclaw/openclaw.git
cd openclaw

# 영속적 상태 저장 디렉토리 생성
mkdir -p ~/.openclaw
mkdir -p ~/.openclaw/workspace
```

Docker 컨테이너는 재시작하면 내부 데이터가 사라집니다. 설정, 메모리, 작업 파일 등은 반드시 호스트 디렉토리에 마운트해서 보존해야 합니다.

### 5단계: 환경 변수 설정

프로젝트 루트에 `.env` 파일을 생성합니다.

```bash
OPENCLAW_IMAGE=openclaw:latest
OPENCLAW_GATEWAY_TOKEN=여기에-강력한-토큰-입력
OPENCLAW_GATEWAY_BIND=lan
OPENCLAW_GATEWAY_PORT=18789
OPENCLAW_CONFIG_DIR=/home/$USER/.openclaw
OPENCLAW_WORKSPACE_DIR=/home/$USER/.openclaw/workspace
GOG_KEYRING_PASSWORD=여기에-강력한-비밀번호-입력
XDG_CONFIG_HOME=/home/node/.openclaw
```

토큰과 비밀번호는 다음 명령어로 안전하게 생성할 수 있습니다.

```bash
openssl rand -hex 32
```

### 6단계: Docker Compose 설정 및 빌드

`docker-compose.yml`을 구성하고 빌드합니다. 여기서 핵심은 **필요한 바이너리를 Docker 이미지에 미리 포함(bake)시키는 것**입니다. 런타임에 설치한 바이너리는 컨테이너 재시작 시 사라지기 때문에 반드시 Dockerfile에서 설치해야 합니다.

```bash
# 빌드 및 실행
docker compose build
docker compose up -d openclaw-gateway

# 게이트웨이 동작 확인
docker compose logs -f openclaw-gateway
```

`[gateway] listening on ws://0.0.0.0:18789` 메시지가 나오면 성공입니다.

### 7단계: 로컬에서 접속

SSH 터널을 통해 로컬 브라우저에서 안전하게 접속할 수 있습니다.

```bash
gcloud compute ssh openclaw-gateway \
  --zone=us-central1-a \
  -- -L 18789:127.0.0.1:18789
```

브라우저에서 `http://127.0.0.1:18789/`을 열고 게이트웨이 토큰을 입력하면 Control UI에 접근됩니다. 여기서 메시징 채널(WhatsApp, Telegram, Discord 등)을 연결하고 AI 모델을 설정하면 개인 에이전트가 바로 동작을 시작합니다.

## 업데이트 방법

OpenClaw를 최신 버전으로 업데이트하는 것은 간단합니다.

```bash
cd ~/openclaw
git pull
docker compose build
docker compose up -d
```

특히 보안 패치는 즉시 적용하는 것이 좋습니다. 2026년 초에 치명적인 LFI 취약점이 발견된 사례가 있었으니, 항상 최신 버전을 유지하세요.

## 비용 정리

클라우드 VM에서 OpenClaw를 운영할 때의 월간 비용을 정리하면 다음과 같습니다.

| 항목 | 비용 |
|------|------|
| GCP e2-small VM | ~$12/월 |
| LLM API (Claude/GPT) | $10~100/월 (사용량에 따라) |
| **합계** | **약 $22~112/월** |

맥미니를 구입하는 경우 초기 비용 80만 원 이상에 전기세가 매월 추가됩니다. 24시간 에이전트 운영만이 목적이라면 클라우드 VM이 훨씬 합리적인 선택입니다.

## 마무리

OpenClaw는 설치 장소보다 **어떤 LLM을 쓰느냐**가 사용 경험을 결정합니다. 맥미니든 클라우드 VM이든 결국 API를 통해 고성능 모델을 호출하는 것은 동일합니다. 그렇다면 별도의 하드웨어를 구매할 이유 없이, 월 12달러짜리 클라우드 VM에 Docker로 깔끔하게 설치하는 것이 현재는 현명한 방법이 아닐까 합니다.

자세한 설정은 [OpenClaw 공식 GCP 가이드](https://docs.clawd.bot/platforms/gcp)를 참고하세요.
