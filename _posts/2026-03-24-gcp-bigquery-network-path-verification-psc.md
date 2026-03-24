---
layout: post
title: "GCP에서 BigQuery 접근 시 내부망 vs 외부망 판별법과 PSC 구성 가이드"
date: 2026-03-24
tags: [google-cloud, bigquery, vpc, private-google-access, psc, private-service-connect, cloud-run, gce, vertex-ai, agent-engine, networking]
---

Cloud Run, GCE, Vertex AI Agent Engine에서 BigQuery API를 호출할 때, 트래픽이 **Google 내부망**을 타는 건지 **외부 인터넷**을 경유하는 건지 궁금해지는 순간이 있습니다. 특히 금융권이나 공공기관 프로젝트에서는 "정말 내부망으로 통신하는 게 맞느냐"는 질문이 반드시 나옵니다.

이 글에서는 **네트워크 경로를 확인하는 방법**, 서비스 유형별 네트워킹 구조, 그리고 **Private Service Connect(PSC)** 구성까지 실전 관점에서 정리합니다.

---

## 1. 내부망인지 확인하는 2가지 방법

### 1-1. DNS 해석 결과로 판별 (가장 간편)

GCE 인스턴스에서 BigQuery API 엔드포인트의 DNS resolve 결과를 보면 힌트를 얻을 수 있습니다.

```bash
# 방법 A: dig로 확인
dig bigquery.googleapis.com

# 방법 B: Python으로 확인
python3 -c "import socket; print(socket.getaddrinfo('bigquery.googleapis.com', 443))"
```

실제 테스트 결과 예시:

```
;; ANSWER SECTION:
bigquery.googleapis.com. 300    IN      A       34.128.10.106
```

```python
[(<AddressFamily.AF_INET: 2>, <SocketKind.SOCK_STREAM: 1>, 6, '', ('34.128.10.106', 443)),
 (<AddressFamily.AF_INET6: 10>, <SocketKind.SOCK_STREAM: 1>, 6, '', ('2600:1900:4250:d::200a', 443, 0, 0)),
 ...]
```

**판별 기준:**

| 해석된 IP 대역 | 의미 |
|---------------|------|
| `199.36.153.4/30` | `restricted.googleapis.com` 경유 → VPC SC + 내부망 |
| `199.36.153.8/30` | `private.googleapis.com` 경유 → 내부망 |
| `142.250.x.x`, `172.217.x.x`, `34.128.x.x` 등 | Google 공인 IP → 외부 경로 가능성 |

위 테스트 결과에서는 `34.128.10.106`과 `2600:1900:4250:d::200a`가 나왔는데, 이는 Google 공인 IP 대역입니다. `restricted`도 `private`도 아닙니다. 즉 **별도의 Private Service Connect나 restricted/private DNS 존 설정이 되어 있지 않다**는 뜻입니다.

### 1-2. VPC Flow Logs로 확인 (가장 확실)

VPC Flow Logs를 켜면 실제 트래픽의 목적지 IP를 볼 수 있습니다.

```bash
# GCE에서 BigQuery API 엔드포인트의 IP 확인
nslookup bigquery.googleapis.com

# Private Google Access가 활성화된 서브넷에서는
# restricted.googleapis.com (199.36.153.4/30) 또는
# private.googleapis.com (199.36.153.8/30) 대역으로 resolve 됨

# 반면 외부 인터넷 경유 시에는 공인 IP로 resolve 됨
```

VPC Flow Logs에서 destination IP가 위의 private 대역이면 내부망을 사용하는 것이 확실합니다.

---

## 2. 공인 IP로 resolve 되면 무조건 외부망인가?

**아닙니다.** 이 부분이 가장 혼동을 주는 포인트입니다.

GCP 공식 문서에 명확한 근거가 있습니다:

> "Packets sent from VMs in your VPC network to Google APIs and services remain within Google's network."
> — [Private Google Access 공식 문서](https://cloud.google.com/vpc/docs/private-google-access)

> "Requests from one Cloud Run resource to another or to other Google Cloud services stay within Google's internal network."
> — [Cloud Run Private Networking 공식 문서](https://cloud.google.com/run/docs/securing/private-networking)

즉, IP 주소가 공개 라우팅 가능(publicly routable)하더라도, **실제 패킷은 Google 내부 네트워크 안에서만 이동합니다.** 인터넷 공중망을 경유하지 않습니다.

경우를 나눠보면:

| 인스턴스 구성 | DNS Resolve 결과 | 실제 경로 |
|-------------|-----------------|----------|
| 외부 IP 없음 + PGA ON | 공인 IP | **Google 내부 백본** (인터넷 미경유) |
| 외부 IP 있음 | 공인 IP | Google Front End(GFE) 통해 접근. 물리적으로는 Google 인프라 내부이나 "VPC 내부망 전용 경로"는 아님 |
| PGA OFF + 외부 IP 없음 | 공인 IP | **접근 불가** (패킷이 나갈 경로 자체가 없음) |

따라서 DNS resolve IP만으로 단정할 수 없고, **인스턴스의 구성을 함께 확인해야** 합니다.

```bash
# 1) 해당 인스턴스에 외부 IP가 있는지
curl -s -H "Metadata-Flavor: Google" \
  http://metadata.google.internal/computeMetadata/v1/instance/network-interfaces/0/access-configs/0/external-ip

# 2) 서브넷에 Private Google Access가 켜져 있는지
gcloud compute networks subnets describe <SUBNET_NAME> \
  --region=<REGION> \
  --format="get(privateIpGoogleAccess)"

# 3) private/restricted DNS 존이 구성되어 있는지
gcloud dns managed-zones list --filter="visibility=private"
```

### 구성별 비교 — 감사(audit) 증명 가능 여부

"인터넷을 경유하지 않는다"는 사실은 구성 수준에 따라 **증명 가능성**이 달라집니다.

| 구성 | 인터넷 경유? | Google 내부 네트워크? | 외부 IP 노출? | 감사 시 증명 가능? |
|-----|-----------|------------------|-------------|-----------------|
| 기본 (default) | ❌ 경유 안 함 | ✅ Google 네트워크 내부 | IP 주소는 공개 라우팅 가능 | 어려움 |
| Private Google Access | ❌ 경유 안 함 | ✅ Google 네트워크 내부 | 외부 IP 불필요 | 부분적 |
| Private Service Connect | ❌ 경유 안 함 | ✅ Google 네트워크 내부 | ❌ 완전 사설 IP | ✅ 감사 가능 |

기본 설정에서도 인터넷을 경유하지 않지만, IP 주소가 공개 라우팅 가능하다는 점 때문에 금융권 등 규제 환경에서는 설명이 어렵습니다. **PSC 엔드포인트를 사용하면** DNS도 내부 해석(`*.googleapis.com` → 내부 IP)으로 완전히 격리되어, 기술적으로 증명할 수 있습니다.

---

## 3. BigQuery 통신 보안 채널 분석

네트워크 경로와 별개로, **통신 채널 자체의 보안**도 중요합니다. BigQuery API 통신은 TLS와 ALTS 이중 보호 구조로 되어 있습니다.

### 통신 경로별 보안 메커니즘

| 경로 | 기술 | 암호화 방식 |
|-----|------|-----------|
| Client → GFE (외부) | TLS (BoringSSL) | HTTPS/TLS 1.2~1.3, Forward Secrecy, FIPS 140-3 Level 1 |
| GFE → BigQuery | ALTS | AES-128-GCM, 서비스 간 인증 포함 |
| Cloud Run/Agent Engine → BigQuery (내부, Private IP) | ALTS | AES-128-GCM, 세션 키 주기적 교체 |

### ALTS (Application Layer Transport Security)

Google 인프라 내부에서 서비스 간 통신을 보호하는 핵심 기술입니다.

- GFE → BigQuery, Cloud Run 백엔드 → BigQuery 모두 ALTS로 보호
- 서비스별 Google internal CA 발급 credential로 **상호 인증(mutual auth)**
- 암호화: AES-128-GCM (기본), 스토리지 레이어는 AES-256
- Handshake: Elliptic Curve + ML-KEM (양자 내성, FIPS 203)
- BoringSSL 또는 PSP로 구현, FIPS 140-2/3 Level 1 검증

### BigQuery API 자체의 보안

- **REST API**: HTTPS 강제 (`http://` 요청 시 "SSL is required" 오류 반환)
- **Storage Read/Write API**: gRPC + TLS, ALTS 캡슐화
- `bigquery.googleapis.com` 엔드포인트는 GFE를 통해 TLS 종료 후 ALTS로 내부 전달

> 참고: [Encryption in transit for Google Cloud](https://cloud.google.com/docs/security/encryption-in-transit), [Google infrastructure security design overview](https://cloud.google.com/docs/security/infrastructure/design)

---

## 4. Private Google Access(PGA) 설정

Private Google Access는 **서브넷 단위**로 설정합니다.

### Console에서 켜기

`VPC network` → `VPC networks` → 해당 VPC 선택 → `Subnets` 탭 → 서브넷 클릭 → `Edit` → **Private Google Access**를 `On`으로 변경 → `Save`

### gcloud CLI로 켜기

```bash
gcloud compute networks subnets update <SUBNET_NAME> \
  --region=<REGION> \
  --enable-private-ip-google-access
```

### 현재 상태 확인

```bash
gcloud compute networks subnets describe <SUBNET_NAME> \
  --region=<REGION> \
  --format="get(privateIpGoogleAccess)"
# True가 나오면 이미 켜져 있음
```

> PGA가 의미를 갖는 건 **외부 IP가 없는 GCE 인스턴스**일 때입니다. 외부 IP가 있는 인스턴스는 이 설정과 무관하게 이미 Google API에 접근 가능합니다. PGA는 외부 IP 없이 내부 전용으로 운영하는 인스턴스가 `googleapis.com` 서비스에 접근할 수 있게 해주는 역할입니다.

---

## 5. 서비스 유형별 네트워킹 구조

### 5-1. GCE (Google Compute Engine)

가장 직관적입니다. 사용자 VPC 안에서 실행되므로, 서브넷의 PGA 설정과 외부 IP 유무에 따라 네트워크 경로가 결정됩니다.

```
GCE 인스턴스 → 서브넷(PGA ON/OFF) → BigQuery API
                  ↓
         외부 IP 없으면 PGA 필요
         외부 IP 있으면 PGA 무관
```

내부망 경로를 명시적으로 강제하고 싶다면, Cloud DNS에서 `googleapis.com`을 `restricted.googleapis.com`(199.36.153.4/30) 또는 `private.googleapis.com`(199.36.153.8/30)으로 매핑하는 **프라이빗 DNS 존**을 생성합니다.

### 5-2. Cloud Run

Cloud Run은 기본적으로 **Google 관리형 인프라**에서 실행됩니다. BigQuery 등 Google API 호출 시 Google 내부 네트워크를 통해 처리됩니다.

사용자 VPC 내 리소스에 접근하려면 **VPC 커넥터**(Serverless VPC Access) 또는 **Direct VPC Egress**를 설정해야 합니다.

| 구성 | Google API 접근 | 사용자 VPC 접근 |
|-----|----------------|----------------|
| 기본 (VPC 커넥터 없음) | Google 내부망 | 불가 |
| VPC 커넥터 사용 | Google 내부망 | 가능 |
| Direct VPC Egress | Google 내부망 | 가능 |

VPC 커넥터를 사용하는 경우 VPC Flow Logs로 트래픽을 확인할 수 있으며, 서브넷의 PGA 설정이 적용됩니다.

### 5-3. Vertex AI Agent Engine

Agent Engine은 **GCE처럼 사용자 VPC 안에서 돌아가는 게 아닙니다.** Google의 **tenant project**에서 실행되기 때문에 사용자가 서브넷을 직접 제어할 수 없습니다.

그래서 "PGA를 켜야 하나?"라는 질문 자체가 적용되지 않습니다.

Agent Engine에는 **3가지 네트워킹 모드**가 있습니다:

| 모드 | 특징 | 인터넷 접근 | 사용자 VPC 접근 |
|------|------|-----------|----------------|
| **Standard** (기본) | 별도 설정 없음 | 가능 | 불가 |
| **VPC Service Controls** | 보안 경계 설정 | **차단** | 불가 |
| **PSC Interface** | 사용자 VPC 연결 | 설정에 따라 | **가능** |

각 모드에서 BigQuery를 호출할 때의 보안 수준:

| 배포 모델 | BigQuery 연결 경로 | 보안 수준 |
|----------|------------------|----------|
| Standard | 공개 API (GFE 경유) | TLS + ALTS (기본 보호) |
| VPC-SC | Private Google Access | 퍼미터 격리 + ALTS, 인터넷 egress 차단 |
| PSC-I | Private Service Connect | VPC 내 완전 사설 경로 + ALTS |

**Standard 모드**에서 BigQuery 등 Google API를 호출하면 Google 내부 네트워크를 통해 처리됩니다. Agent Engine 자체가 Google 인프라 안에서 돌아가기 때문에, "내부망이냐 외부망이냐" 구분 자체가 GCE와는 다른 맥락입니다.

---

## 6. Agent Engine PSC Interface 구성 가이드

에이전트가 사용자 VPC 내의 프라이빗 리소스(온프레미스 DB, Cloud SQL private IP 등)에 접근해야 할 때 PSC Interface를 구성합니다. 설정은 **고객 VPC 구성 → IAM 권한 → DNS 피어링 → Agent Engine 배포** 4개 영역에서 진행됩니다.

### 아키텍처 흐름

```
Agent Engine (Google-managed tenant VPC)
    ↓  PSC Interface (eth1)
Network Attachment (고객 VPC intf-subnet: 192.168.10.0/28)
    ↓  내부 라우팅
Proxy VM (10.10.10.2) ← DNS: proxy-vm.demo.com
    ↓  Cloud NAT (인터넷 액세스 필요 시)
사설 리소스 또는 인터넷
```

Agent Engine은 PSC-I 구성 시 **직접 인터넷 경로가 없습니다.** 에이전트가 외부 API를 호출해야 하면 Proxy VM을 통해 우회해야 합니다.

### Step 1: API 활성화

```bash
gcloud services enable compute.googleapis.com
gcloud services enable aiplatform.googleapis.com
gcloud services enable dns.googleapis.com
gcloud services enable storage.googleapis.com
```

### Step 2: VPC 및 서브넷 생성

**두 개의 서브넷**이 필요합니다. 용도가 다릅니다.

```bash
# VPC 생성 (기존 VPC 사용 시 생략)
gcloud compute networks create consumer-vpc --subnet-mode=custom

# 서브넷 1: Proxy VM 용 (사설 리소스 또는 인터넷 egress)
gcloud compute networks subnets create rfc1918-subnet1 \
  --range=10.10.10.0/28 \
  --network=consumer-vpc \
  --region=us-central1

# 서브넷 2: PSC Network Attachment 전용 (/28 최소)
gcloud compute networks subnets create intf-subnet \
  --range=192.168.10.0/28 \
  --network=consumer-vpc \
  --region=us-central1
```

> `intf-subnet`은 **PSC 전용**으로 다른 리소스와 공유하지 않도록 합니다. Agent Engine은 `max_instances × 2`개의 IP를 이 서브넷에서 할당합니다.

### Step 3: PSC Network Attachment 생성

Agent Engine이 이 Attachment를 통해 고객 VPC에 연결됩니다.

```bash
gcloud compute network-attachments create psc-network-attachment \
  --region=us-central1 \
  --connection-preference=ACCEPT_AUTOMATIC \
  --subnets=intf-subnet

# 생성 확인
gcloud compute network-attachments describe psc-network-attachment \
  --region=us-central1
```

`ACCEPT_AUTOMATIC`은 Vertex AI P4SA가 자동으로 연결을 수락합니다. 수동 승인이 필요하면 `ACCEPT_MANUAL`을 사용합니다.

### Step 4: Proxy VM + Cloud NAT 구성 (인터넷 egress 필요 시)

에이전트가 외부 API를 호출해야 하는 경우에만 필요합니다. BigQuery 등 Google API만 호출한다면 이 단계는 생략 가능합니다.

```bash
# Cloud Router + Cloud NAT 생성 (Proxy VM의 인터넷 egress용)
gcloud compute routers create cloud-router-for-nat \
  --network=consumer-vpc --region=us-central1

gcloud compute routers nats create cloud-nat-us-central1 \
  --router=cloud-router-for-nat \
  --auto-allocate-nat-external-ips \
  --nat-all-subnet-ip-ranges \
  --region=us-central1

# Proxy VM 생성 (tinyproxy 설치)
gcloud compute instances create proxy-vm \
  --machine-type=e2-micro \
  --image-family=debian-11 \
  --image-project=debian-cloud \
  --no-address \
  --can-ip-forward \
  --zone=us-central1-a \
  --subnet=rfc1918-subnet1 \
  --metadata=startup-script='apt-get update && apt-get install -y tinyproxy'
```

Proxy VM에 SSH 접속 후 `/etc/tinyproxy/tinyproxy.conf`를 수정합니다:

```
Listen 10.10.10.2
Allow 192.168.10.0/24   # PSC 서브넷 허용
```

```bash
sudo systemctl restart tinyproxy
```

### Step 5: 방화벽 규칙 설정

PSC Interface 서브넷에서 Proxy VM 서브넷으로의 ingress를 허용합니다.

```bash
gcloud compute firewall-rules create allow-psc-to-proxy \
  --network=consumer-vpc \
  --action=ALLOW \
  --rules=ALL \
  --direction=INGRESS \
  --source-ranges="192.168.10.0/28" \
  --destination-ranges="10.10.10.0/28"
```

### Step 6: Cloud DNS Private Zone + A 레코드

에이전트가 FQDN으로 VPC 내 리소스에 접근하려면 Cloud DNS private zone이 필요합니다.

```bash
# Private DNS Zone 생성
gcloud dns managed-zones create private-dns-zone \
  --dns-name="demo.com." \
  --visibility=private \
  --networks="consumer-vpc"

# Proxy VM IP 확인
gcloud compute instances describe proxy-vm \
  --zone=us-central1-a | grep networkIP

# A 레코드 추가 (IP는 실제 값으로 교체)
gcloud dns record-sets create proxy-vm.demo.com. \
  --zone=private-dns-zone \
  --type=A \
  --ttl=300 \
  --rrdatas="10.10.10.2"
```

### Step 7: IAM 권한 부여

Vertex AI Service Agent에 **두 가지 권한**이 필요합니다.

```bash
PROJECT_NUMBER=$(gcloud projects describe $PROJECT_ID --format="value(projectNumber)")

# Vertex AI Service Agent 생성 (없으면)
gcloud beta services identity create \
  --service=aiplatform.googleapis.com \
  --project=$PROJECT_NUMBER

# 1) Network Attachment 수정 권한
gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member="serviceAccount:service-${PROJECT_NUMBER}@gcp-sa-aiplatform.iam.gserviceaccount.com" \
  --role="roles/compute.networkAdmin"

# 2) DNS 피어링 권한
gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member="serviceAccount:service-${PROJECT_NUMBER}@gcp-sa-aiplatform.iam.gserviceaccount.com" \
  --role="roles/dns.peer"
```

> `roles/compute.networkAdmin` 대신 최소 권한으로 Custom Role(`compute.networkAttachments.get`, `compute.networkAttachments.update`, `compute.regionOperations.get`)을 사용할 수도 있습니다.

### Step 8: Agent Engine 배포

배포 시 `psc_interface_config`를 지정합니다. 두 가지 방식이 있습니다.

**방식 A — ADK 설정 파일 (.agent_engine_config.json)**

```python
import json, os

config_data = {
    "requirements": [
        "google-cloud-aiplatform[agent_engines,adk]",
    ],
    "psc_interface_config": {
        "network_attachment": f"projects/{PROJECT_ID}/regions/us-central1/networkAttachments/psc-network-attachment",
        "dns_peering_configs": [
            {
                "domain": "demo.com.",
                "target_project": PROJECT_ID,
                "target_network": "consumer-vpc"
            }
        ]
    }
}

os.makedirs("my_agent", exist_ok=True)
with open("my_agent/.agent_engine_config.json", "w") as f:
    json.dump(config_data, f, indent=4)
```

**방식 B — Python SDK 직접 호출**

```python
from google.cloud import aiplatform

client = aiplatform.Client(project="my-project", location="us-central1")

remote_agent = client.agent_engines.create(
    agent=local_agent,
    config={
        "psc_interface_config": {
            "network_attachment": "projects/my-project/regions/us-central1/networkAttachments/psc-network-attachment",
            "dns_peering_configs": [
                {
                    "domain": "demo.com.",
                    "target_project": "my-project",
                    "target_network": "consumer-vpc",
                },
            ],
        },
    },
)
```

**방식 C — REST API 직접 호출**

```python
import requests

ENDPOINT = "https://us-central1-aiplatform.googleapis.com"

response = requests.post(
    f"{ENDPOINT}/v1beta1/projects/{PROJECT_ID}/locations/us-central1/reasoningEngines",
    headers={"Authorization": f"Bearer {token}"},
    json={
        "displayName": "My Private Agent",
        "spec": {
            "packageSpec": {
                "pickleObjectGcsUri": f"gs://{BUCKET}/agent.pkl",
                "requirementsGcsUri": f"gs://{BUCKET}/requirements.txt",
                "pythonVersion": "3.10"
            },
            "deploymentSpec": {
                "pscInterfaceConfig": {
                    "networkAttachment": f"projects/{PROJECT_ID}/regions/us-central1/networkAttachments/psc-network-attachment",
                    "dnsPeeringConfigs": [
                        {
                            "domain": "demo.com.",
                            "targetProject": PROJECT_ID,
                            "targetNetwork": "consumer-vpc"
                        }
                    ]
                }
            }
        }
    }
)
```

> `network_attachment`에는 이름만 넣어도 되지만, Shared VPC 등 Agent Engine을 사용하는 프로젝트와 network attachment가 다른 프로젝트에 있는 경우 full path를 넣어야 합니다.

### 전체 설정 요약

| 단계 | 위치 | 주요 리소스 | 비고 |
|-----|------|-----------|------|
| API 활성화 | 고객 프로젝트 | compute, aiplatform, dns | - |
| VPC/서브넷 | 고객 프로젝트 | consumer-vpc, intf-subnet(/28 이상) | PSC 전용 서브넷 분리 |
| Network Attachment | 고객 프로젝트 | psc-network-attachment | 핵심 리소스 |
| Proxy VM | 고객 VPC | proxy-vm + tinyproxy | 인터넷 egress 필요 시만 |
| Cloud NAT | 고객 프로젝트 | Cloud Router + NAT | Proxy VM 인터넷 egress |
| 방화벽 | 고객 VPC | PSC 서브넷 → Proxy 서브넷 허용 | - |
| Cloud DNS | 고객 프로젝트 | Private Zone + A 레코드 | FQDN 접근 시 |
| IAM | 고객 프로젝트 | Vertex AI SA → networkAdmin + dns.peer | - |
| Agent 배포 | Agent Engine | pscInterfaceConfig 지정 | 배포 시 1회 설정 |

### Shared VPC 환경에서의 권장 구성

Network Attachment는 **Vertex AI 서비스 프로젝트**(Agent Engine 배포 프로젝트)에 생성하는 것을 권장합니다. Vertex AI P4SA가 해당 프로젝트에서 Attachment를 패치하는 방식이므로 권한 관리가 단순해집니다.

여러 에이전트가 하나의 network attachment를 공유하도록 구성할 수도 있고, 각각 전용 network attachment를 사용하도록 할 수도 있습니다.

---

## 7. 서비스별 BigQuery 접근 경로 비교 정리

| 항목 | GCE | Cloud Run | Agent Engine |
|------|-----|-----------|-------------|
| **실행 환경** | 사용자 VPC 내 | Google 관리형 (VPC 커넥터로 연결 가능) | Google tenant project |
| **BigQuery 접근** | PGA + 서브넷 설정 | 기본적으로 내부망 | 기본적으로 내부망 |
| **PGA 설정 필요** | 외부 IP 없는 경우 필요 | VPC 커넥터 사용 시 서브넷에 설정 | 해당 없음 |
| **내부망 강제** | private DNS 존 구성 | VPC 커넥터 + private DNS 존 | VPC-SC 또는 PSC-I 모드 |
| **사용자 VPC 접근** | 기본 가능 | VPC 커넥터 또는 Direct VPC Egress | PSC Interface 구성 필요 |
| **네트워크 확인** | dig, VPC Flow Logs | VPC 커넥터 사용 시 Flow Logs | Google 관리 영역이라 직접 확인 불가 |

---

## 8. 내부망 경로를 명시적으로 보장하려면

DNS resolve 결과가 공인 IP로 나오더라도 PGA가 켜져 있으면 실질적으로 내부망이지만, 이것만으로는 감사(audit) 시 설명이 어려울 수 있습니다. 명시적으로 내부망을 보장하려면:

### Cloud DNS Private Zone 구성

`googleapis.com`을 restricted 또는 private 대역으로 매핑하는 DNS 존을 생성합니다.

```bash
# restricted.googleapis.com으로 매핑 (VPC-SC 사용 시)
gcloud dns managed-zones create googleapis-restricted \
  --dns-name=googleapis.com \
  --visibility=private \
  --networks=my-vpc \
  --description="Route googleapis.com to restricted VIPs"

gcloud dns record-sets create googleapis.com \
  --zone=googleapis-restricted \
  --type=CNAME \
  --ttl=300 \
  --rrdatas="restricted.googleapis.com."

gcloud dns record-sets create restricted.googleapis.com \
  --zone=googleapis-restricted \
  --type=A \
  --ttl=300 \
  --rrdatas="199.36.153.4,199.36.153.5,199.36.153.6,199.36.153.7"
```

이렇게 설정하면 `dig bigquery.googleapis.com` 결과가 `199.36.153.4/30` 대역으로 나오게 되어, DNS 레벨에서도 내부망 사용을 확인할 수 있습니다.

---

## 빠른 판별 플로차트

```
BigQuery API 호출 시 내부망인가?
│
├── GCE의 경우
│   ├── 외부 IP 없음 + PGA ON → ✅ 내부망
│   ├── 외부 IP 있음 → ⚠️ Google 인프라 내부이나 전용 경로는 아님
│   └── PGA OFF + 외부 IP 없음 → ❌ 접근 불가
│
├── Cloud Run의 경우
│   ├── 기본 (VPC 커넥터 없음) → ✅ Google 내부망
│   └── VPC 커넥터 사용 → ✅ 내부망 (서브넷 PGA 설정 적용)
│
└── Agent Engine의 경우
    ├── Standard 모드 → ✅ Google 내부망
    ├── VPC-SC 모드 → ✅ 내부망 + 인터넷 차단
    └── PSC-I 모드 → ✅ 내부망 + 사용자 VPC 연결
```

---

## 마무리

"BigQuery에 접근할 때 내부망을 통하는가?"라는 질문의 답은 **서비스 유형과 구성에 따라 다릅니다.** GCE는 서브넷과 PGA 설정이 핵심이고, Cloud Run과 Agent Engine은 Google 관리형 서비스이기 때문에 기본적으로 Google 내부 네트워크를 사용합니다.

DNS resolve 결과가 공인 IP로 나온다고 해서 반드시 외부망은 아니라는 점, 그리고 명시적으로 내부망을 보장하려면 Cloud DNS Private Zone이나 VPC-SC를 구성해야 한다는 점을 기억하면 됩니다.

통신 채널 보안 측면에서는, BigQuery API 통신이 TLS + ALTS 이중 보호 구조로 되어 있어 기본적으로 안전합니다. 다만 금융권 등 규제 환경에서는 이것만으로 충분하지 않을 수 있으므로, VPC-SC 또는 PSC를 추가하여 완전 사설 채널로 격리하는 것이 권장됩니다.

Agent Engine에서 사용자 VPC 내부 리소스에 접근해야 하는 경우에는 PSC Interface를 구성하면 되고, 단순히 BigQuery 등 Google API만 호출하는 경우라면 별도 네트워크 설정 없이 Standard 모드로 충분합니다.

> **한줄 요약:** BigQuery와의 통신은 기본적으로 Google 내부망 + TLS + ALTS 이중 보호 구조이며, 고보안 환경이라면 VPC-SC 또는 PSC-I를 추가하여 완전 사설 채널로 격리하고 감사 시 증명 가능한 구조를 갖추는 것이 권장됩니다.
