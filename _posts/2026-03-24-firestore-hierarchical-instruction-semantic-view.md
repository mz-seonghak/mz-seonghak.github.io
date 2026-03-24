---
layout: post
title: "Firestore로 AI Agent Instruction과 Semantic View를 계층적으로 관리하는 설계"
date: 2026-03-24
tags: [firestore, ai-agent, semantic-view, instruction, architecture, gcp, multi-tenant]
---

AI Agent 기반 시스템을 운영하다 보면, **"이 에이전트에게 어떤 지시(Instruction)를 줄 것인가"**와 **"어떤 데이터를 어떤 관점(Semantic View)으로 볼 것인가"**를 체계적으로 관리해야 하는 시점이 옵니다.

특히 기업 환경에서는 개인이 만든 설정을 팀에 공유하고, 검증된 것을 전사 표준으로 승격시키는 **계층적 관리 구조**가 필수입니다.

이 글에서는 Firestore를 활용하여 Instruction과 Semantic View를 **사용자 → 부서 → 글로벌** 3단계로 관리하고, 특정 에이전트에 배정하는 구조에 대해서 설명합니다.

가상의 기업 SH은행을 예로 들어 구체적인 설계 방법을 살펴보겠습니다.

---

## 1. 왜 계층적 관리가 필요한가

AI Agent 시스템에서 Instruction과 Semantic View를 단순히 1:1로 관리하면 다음과 같은 문제가 발생합니다.

| 문제 | 예시 |
|------|------|
| 중복 작업 | 같은 팀원 5명이 비슷한 Instruction을 각자 작성 |
| 품질 불균형 | 숙련자의 노하우가 개인에게만 존재 |
| 표준 부재 | 부서마다 다른 Semantic View로 동일 데이터를 다르게 해석 |
| 관리 불가 | 수백 개의 개인 설정이 산재하여 어떤 것이 검증된 것인지 파악 불가 |

이를 해결하기 위한 핵심 개념이 **3단계 계층 구조**입니다.

```
글로벌 (Global)         ← 전사 표준. 관리자가 배포
  ↑ 승격(Promote)
부서 (Department)       ← 팀 내 공유. 팀 리더가 관리
  ↑ 승격(Promote)
사용자 (User)           ← 개인 작업 공간. 자유롭게 실험
```

---

## 2. 관리 대상 정의

설계에 앞서, 이 시스템이 관리하는 두 가지 핵심 리소스를 명확히 정의합니다.

### 2-1. Instruction (에이전트 지시)

AI Agent에게 전달하는 행동 지침입니다. 시스템 프롬프트, 응답 규칙, 제약 조건 등을 포함합니다.

```yaml
name: "금융 상담 에이전트 지시"
content: |
  당신은 SH은행의 금융 상담 AI 어시스턴트입니다.
  - 고객 질문에 정확하고 친절하게 답변합니다.
  - 투자 권유는 하지 않습니다.
  - 금리 정보는 반드시 최신 기준으로 안내합니다.
  - 답변 끝에 "추가 문의 사항이 있으시면 말씀해 주세요"를 붙입니다.
```

### 2-2. Semantic View (시맨틱 뷰)

데이터를 특정 관점으로 바라보는 가상의 뷰 정의입니다. 에이전트가 데이터에 접근할 때 어떤 테이블을 어떤 관점으로 조회할지를 YAML로 기술합니다.

```yaml
name: "SV_예금잔액"
description: "일별 전체 예금 잔액 현황"
base_table: "banking.daily_deposit_balance"
columns:
  - name: base_date
    description: "기준일자"
  - name: total_balance
    description: "총 예금잔액 (원)"
filters:
  - "base_date >= CURRENT_DATE - 30"
```

---

## 3. Firestore 컬렉션 설계

Firestore의 특성(문서 기반, 서브컬렉션, 컬렉션 그룹 쿼리, 경로 기반 보안 규칙)을 최대한 활용하는 **계층적 컬렉션 구조**를 채택합니다.

### 3-1. 전체 구조

```
[Firestore]
│
├── instructions/                              ← Instruction 최상위 컬렉션
│   ├── _global/                               ← 글로벌 scope
│   │     └── workspaces/
│   │           └── 표준_상담/
│   │                 └── items/
│   │                       ├── 금융상담_기본
│   │                       └── 리스크_안내
│   │
│   ├── dept:IT팀/                             ← 부서 scope
│   │     └── workspaces/
│   │           └── IT_운영/
│   │                 └── items/
│   │                       └── 장애대응_가이드
│   │
│   └── user:hong@sh-bank.com/                      ← 사용자 scope
│         └── workspaces/
│               ├── 내_실험/
│               │     └── items/
│               │           ├── 테스트_지시_v1
│               │           └── 테스트_지시_v2
│               └── 상담_커스텀/
│                     └── items/
│                           └── VIP_상담_지시
│
├── semantic_views/                            ← Semantic View 최상위 컬렉션
│   ├── _global/
│   │     └── workspaces/
│   │           └── 표준_KPI/
│   │                 └── views/
│   │                       ├── SV_예금잔액
│   │                       └── SV_고객수
│   │
│   ├── dept:IT팀/
│   │     └── workspaces/
│   │           └── IT_대시보드/
│   │                 └── views/
│   │                       └── SV_서버현황
│   │
│   └── user:hong@sh-bank.com/
│         └── workspaces/
│               ├── 예금분석/
│               │     └── views/
│               │           ├── SV_예금잔액
│               │           └── SV_예금추이
│               └── 대출리포트/
│                     └── views/
│                           └── SV_연체현황
│
└── agent_assignments/                         ← 에이전트 배정 컬렉션
      └── {agent_id}/
            └── config
                  ├── instruction_ref: "..."
                  └── semantic_view_refs: [...]
```

### 3-2. 경로 패턴

두 리소스 모두 동일한 경로 패턴을 따릅니다.

```
{resource_type}/{scope}/workspaces/{workspace_id}/{item_collection}/{item_id}
```

| 세그먼트 | 설명 | 예시 |
|----------|------|------|
| `resource_type` | 최상위 컬렉션 | `instructions`, `semantic_views` |
| `scope` | 계층 식별자 | `_global`, `dept:IT팀`, `user:hong@sh-bank.com` |
| `workspace_id` | 워크스페이스(세트) 이름 | `표준_KPI`, `예금분석` |
| `item_collection` | 아이템 서브컬렉션 | `items`(Instruction), `views`(Semantic View) |
| `item_id` | 개별 아이템 | `금융상담_기본`, `SV_예금잔액` |

### 3-3. Scope 문서 ID 규칙

| Scope | 문서 ID 패턴 | 예시 |
|-------|-------------|------|
| 글로벌 | `_global` | `_global` |
| 부서 | `dept:{dept_id}` | `dept:IT팀` |
| 사용자 | `user:{email}` | `user:hong@sh-bank.com` |

`_global`은 언더스코어 접두사로 항상 정렬 최상단에 위치합니다. `dept:`와 `user:` 접두사로 scope 종류를 경로만 보고 즉시 식별할 수 있습니다.

---

## 4. 문서 스키마 설계

### 4-1. Workspace (워크스페이스) 문서

사용자가 관련 리소스를 논리적으로 묶는 단위입니다.

```python
# instructions/{scope}/workspaces/{workspace_id}
# semantic_views/{scope}/workspaces/{workspace_id}
{
    "name": "예금 분석",
    "description": "예금 관련 시맨틱뷰 모음",
    "owner_id": "hong@sh-bank.com",
    "dept_id": "IT팀",
    "scope": "user",               # "global" | "dept" | "user"
    "tags": ["예금", "분석"],
    "item_count": 3,               # 비정규화: 목록 화면에서 건수 표시용
    "created_at": "2026-03-24T09:00:00Z",
    "updated_at": "2026-03-24T14:30:00Z"
}
```

### 4-2. Instruction 문서

```python
# instructions/{scope}/workspaces/{workspace_id}/items/{item_id}
{
    "name": "금융상담 기본 지시",
    "description": "일반 고객 대상 금융 상담 에이전트용 기본 Instruction",
    "content": "당신은 SH은행의 금융 상담 AI 어시스턴트입니다...",
    "content_type": "text",        # "text" | "yaml" | "json"
    "owner_id": "hong@sh-bank.com",
    "workspace_id": "표준_상담",   # 역참조 (collection_group 쿼리용)
    "scope": "global",
    "visibility": "published",     # "draft" | "shared" | "published"
    "promoted_from": "",           # 승격 전 원본 경로
    "assigned_agents": [           # 이 Instruction을 사용 중인 에이전트 목록
        "agent:financial-advisor",
        "agent:loan-consultant"
    ],
    "tags": ["상담", "금융", "기본"],
    "version": 3,
    "created_at": "2026-03-24T09:00:00Z",
    "updated_at": "2026-03-24T14:30:00Z"
}
```

### 4-3. Semantic View 문서

```python
# semantic_views/{scope}/workspaces/{workspace_id}/views/{view_id}
{
    "name": "SV_예금잔액",
    "description": "일별 전체 예금 잔액 현황",
    "yaml_string": "base_table: banking.daily_deposit_balance\n...",
    "owner_id": "hong@sh-bank.com",
    "workspace_id": "예금분석",
    "scope": "user",
    "visibility": "draft",
    "promoted_from": "",
    "tags": ["예금", "KPI"],
    "version": 1,
    "created_at": "2026-03-24T09:00:00Z",
    "updated_at": "2026-03-24T14:30:00Z"
}
```

### 4-4. Agent Assignment 문서

에이전트에 Instruction과 Semantic View를 배정하는 구조입니다.

```python
# agent_assignments/{agent_id}/config
{
    "agent_id": "financial-advisor",
    "agent_name": "금융 상담 에이전트",
    "instruction_ref": "instructions/_global/workspaces/표준_상담/items/금융상담_기본",
    "semantic_view_refs": [
        "semantic_views/_global/workspaces/표준_KPI/views/SV_예금잔액",
        "semantic_views/dept:IT팀/workspaces/IT_대시보드/views/SV_서버현황"
    ],
    "assigned_by": "hong@sh-bank.com",
    "override_instruction_ref": "",  # 사용자별 오버라이드 (선택)
    "is_active": true,
    "updated_at": "2026-03-24T14:30:00Z"
}
```

---

## 5. 쿼리 패턴

Firestore의 경로 기반 접근과 `collection_group` 쿼리를 조합하면, 다양한 조회 시나리오를 효율적으로 처리할 수 있습니다.

### 5-1. 기본 CRUD 쿼리

```python
from google.cloud import firestore

db = firestore.Client()

# ── Instruction 쿼리 ──

# 1) 내 워크스페이스 목록
my_workspaces = db.collection(
    "instructions/user:hong@sh-bank.com/workspaces"
).stream()

# 2) 특정 워크스페이스의 Instruction 목록
my_instructions = db.collection(
    "instructions/user:hong@sh-bank.com/workspaces/내_실험/items"
).stream()

# 3) 글로벌 Instruction 전체
global_instructions = db.collection(
    "instructions/_global/workspaces/표준_상담/items"
).stream()

# 4) 부서 Instruction 전체
dept_instructions = db.collection(
    "instructions/dept:IT팀/workspaces/IT_운영/items"
).stream()
```

### 5-2. 컬렉션 그룹 쿼리

`collection_group`을 사용하면 모든 scope의 같은 이름의 서브컬렉션을 한 번에 검색할 수 있습니다.

```python
# 5) 전체 Instruction 검색 (scope 무관)
all_instructions = db.collection_group("items").stream()

# 6) 전체 Semantic View 검색 (scope 무관)
all_views = db.collection_group("views").stream()

# 7) 특정 태그가 포함된 Instruction 검색
tagged = db.collection_group("items") \
    .where("tags", "array_contains", "상담") \
    .stream()

# 8) 특정 사용자가 만든 모든 Instruction (scope 무관)
user_items = db.collection_group("items") \
    .where("owner_id", "==", "hong@sh-bank.com") \
    .stream()
```

### 5-3. 내가 볼 수 있는 모든 리소스 조합

사용자가 접근 가능한 리소스는 **글로벌 + 내 부서 + 내 개인** 3가지 scope를 병합합니다.

```python
def get_visible_instructions(user_email: str, dept_id: str) -> list:
    """사용자가 접근 가능한 모든 Instruction을 반환"""
    sources = [
        f"instructions/_global/workspaces",
        f"instructions/dept:{dept_id}/workspaces",
        f"instructions/user:{user_email}/workspaces",
    ]

    results = []
    for source in sources:
        workspaces = db.collection(source).stream()
        for ws in workspaces:
            items = db.collection(f"{source}/{ws.id}/items").stream()
            results.extend(items)

    return results


visible = get_visible_instructions("hong@sh-bank.com", "IT팀")
```

---

## 6. 에이전트 배정 구조

이 설계의 핵심 기능 중 하나는 **사용자가 자신의 Instruction을 특정 에이전트에 배정**하는 것입니다.

### 6-1. 배정 모델

```
┌─────────────┐         ┌─────────────┐         ┌─────────────┐
│ Instruction │ ──1:N──▶│  Assignment  │◀──N:1── │    Agent    │
└─────────────┘         └─────────────┘         └─────────────┘
                              │
                              │ N:M
                              ▼
                        ┌─────────────┐
                        │Semantic View│
                        └─────────────┘
```

하나의 에이전트는 하나의 Instruction과 여러 Semantic View를 가질 수 있습니다. 사용자별로 같은 에이전트에 다른 Instruction을 배정할 수 있도록 **사용자별 배정 문서**를 분리합니다.

### 6-2. 사용자별 에이전트 배정

```
agent_assignments/
  └── user:hong@sh-bank.com/                    ← 사용자별 배정
        └── agents/
              ├── financial-advisor/        ← 에이전트별 설정
              │     instruction_ref: "instructions/user:hong@sh-bank.com/..."
              │     semantic_view_refs: [...]
              │     use_global_fallback: true
              │
              └── loan-consultant/
                    instruction_ref: "instructions/dept:IT팀/..."
                    semantic_view_refs: [...]
                    use_global_fallback: false
```

### 6-3. Instruction 해석 우선순위

에이전트가 실행될 때, Instruction을 어디서 가져올지 **우선순위 체인**으로 결정합니다.

```
1. 사용자별 배정 (user assignment)     ← 최우선
2. 부서별 기본값 (dept default)
3. 글로벌 기본값 (global default)      ← 폴백
```

```python
def resolve_instruction(agent_id: str, user_email: str, dept_id: str) -> dict:
    """우선순위에 따라 에이전트의 Instruction을 해석"""

    # 1. 사용자별 배정 확인
    user_assignment = db.document(
        f"agent_assignments/user:{user_email}/agents/{agent_id}"
    ).get()

    if user_assignment.exists:
        ref = user_assignment.to_dict().get("instruction_ref")
        if ref:
            doc = db.document(ref).get()
            if doc.exists:
                return doc.to_dict()

    # 2. 부서 기본값 확인
    dept_assignment = db.document(
        f"agent_assignments/dept:{dept_id}/agents/{agent_id}"
    ).get()

    if dept_assignment.exists:
        ref = dept_assignment.to_dict().get("instruction_ref")
        if ref:
            doc = db.document(ref).get()
            if doc.exists:
                return doc.to_dict()

    # 3. 글로벌 기본값
    global_assignment = db.document(
        f"agent_assignments/_global/agents/{agent_id}"
    ).get()

    if global_assignment.exists:
        ref = global_assignment.to_dict().get("instruction_ref")
        if ref:
            doc = db.document(ref).get()
            if doc.exists:
                return doc.to_dict()

    return None
```

### 6-4. 배정 API 예시

```python
def assign_instruction_to_agent(
    user_email: str,
    agent_id: str,
    instruction_path: str,
    semantic_view_paths: list[str] = None
):
    """사용자가 자신의 Instruction을 특정 에이전트에 배정"""
    doc_ref = db.document(
        f"agent_assignments/user:{user_email}/agents/{agent_id}"
    )

    data = {
        "agent_id": agent_id,
        "instruction_ref": instruction_path,
        "semantic_view_refs": semantic_view_paths or [],
        "assigned_by": user_email,
        "updated_at": firestore.SERVER_TIMESTAMP,
    }

    doc_ref.set(data, merge=True)


# 사용 예시
assign_instruction_to_agent(
    user_email="hong@sh-bank.com",
    agent_id="financial-advisor",
    instruction_path="instructions/user:hong@sh-bank.com/workspaces/상담_커스텀/items/VIP_상담_지시",
    semantic_view_paths=[
        "semantic_views/_global/workspaces/표준_KPI/views/SV_예금잔액",
        "semantic_views/user:hong@sh-bank.com/workspaces/예금분석/views/SV_예금추이",
    ]
)
```

---

## 7. 승격(Promote) 흐름

개인이 만든 리소스를 부서 또는 글로벌로 승격시키는 워크플로입니다.

### 7-1. 승격 단계

```
user:hong@sh-bank.com/workspaces/내_실험/items/VIP_상담_지시
        │
        │  ① 부서 공유 (promote to dept)
        ▼
dept:IT팀/workspaces/공유_지시/items/VIP_상담_지시
        │
        │  ② 글로벌 배포 (promote to global)
        ▼
_global/workspaces/표준_상담/items/VIP_상담_지시
```

### 7-2. 승격 구현

```python
def promote_instruction(
    source_path: str,
    target_scope: str,
    target_workspace: str,
    promoted_by: str
):
    """Instruction을 상위 scope로 승격(복사)"""
    source_doc = db.document(source_path).get()
    if not source_doc.exists:
        raise ValueError(f"소스 문서를 찾을 수 없습니다: {source_path}")

    data = source_doc.to_dict()
    item_id = source_path.split("/")[-1]

    target_path = f"instructions/{target_scope}/workspaces/{target_workspace}/items/{item_id}"

    data.update({
        "promoted_from": source_path,
        "promoted_by": promoted_by,
        "scope": "global" if target_scope == "_global" else "dept",
        "visibility": "published",
        "promoted_at": firestore.SERVER_TIMESTAMP,
    })

    db.document(target_path).set(data)

    # 원본에 승격 이력 기록
    db.document(source_path).update({
        "promoted_to": target_path,
        "promoted_at": firestore.SERVER_TIMESTAMP,
    })

    return target_path


# 사용 예시: 개인 → 부서로 승격
promote_instruction(
    source_path="instructions/user:hong@sh-bank.com/workspaces/내_실험/items/VIP_상담_지시",
    target_scope="dept:IT팀",
    target_workspace="공유_지시",
    promoted_by="hong@sh-bank.com"
)
```

---

## 8. Firestore 보안 규칙

경로 기반 구조의 장점은 **보안 규칙을 간결하게 작성**할 수 있다는 것입니다.

```javascript
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {

    // Instruction 보안 규칙
    match /instructions/{scope}/workspaces/{wsId}/items/{itemId} {

      // 글로벌: 누구나 읽기 가능, 관리자만 쓰기
      allow read: if scope == '_global';
      allow write: if scope == '_global'
                   && request.auth.token.role == 'admin';

      // 부서: 같은 부서원만 읽기, 팀 리더만 쓰기
      allow read: if scope.matches('dept:.*')
                  && request.auth.token.dept_id == scope.split(':')[1];
      allow write: if scope.matches('dept:.*')
                   && request.auth.token.dept_id == scope.split(':')[1]
                   && request.auth.token.is_team_lead == true;

      // 사용자: 본인만 읽기/쓰기
      allow read, write: if scope.matches('user:.*')
                         && request.auth.token.email == scope.split(':')[1];
    }

    // Semantic View도 동일한 패턴 적용
    match /semantic_views/{scope}/workspaces/{wsId}/views/{viewId} {
      allow read: if scope == '_global';
      allow write: if scope == '_global'
                   && request.auth.token.role == 'admin';

      allow read: if scope.matches('dept:.*')
                  && request.auth.token.dept_id == scope.split(':')[1];
      allow write: if scope.matches('dept:.*')
                   && request.auth.token.dept_id == scope.split(':')[1]
                   && request.auth.token.is_team_lead == true;

      allow read, write: if scope.matches('user:.*')
                         && request.auth.token.email == scope.split(':')[1];
    }

    // 에이전트 배정: 본인 배정만 수정 가능
    match /agent_assignments/user:{userEmail}/agents/{agentId} {
      allow read, write: if request.auth.token.email == userEmail;
    }
    match /agent_assignments/dept:{deptId}/agents/{agentId} {
      allow read: if request.auth.token.dept_id == deptId;
      allow write: if request.auth.token.dept_id == deptId
                   && request.auth.token.is_team_lead == true;
    }
    match /agent_assignments/_global/agents/{agentId} {
      allow read: if true;
      allow write: if request.auth.token.role == 'admin';
    }
  }
}
```

경로 자체에 scope 정보가 담겨 있으므로, 문서 내부 필드를 검사하는 복잡한 조건 없이도 접근 제어가 가능합니다.

---

## 9. 전체 워크플로 시나리오

실제 사용 흐름을 시나리오로 정리합니다.

### 시나리오: 홍길동이 Instruction을 만들어 에이전트에 배정하는 전체 과정

```
[1단계] 개인 워크스페이스 생성
  └─ POST instructions/user:hong@sh-bank.com/workspaces/상담_커스텀

[2단계] Instruction 작성
  └─ POST .../상담_커스텀/items/VIP_상담_지시
     { content: "VIP 고객 전용 상담 시 존칭 사용..." }

[3단계] 에이전트에 배정
  └─ PUT agent_assignments/user:hong@sh-bank.com/agents/financial-advisor
     { instruction_ref: ".../VIP_상담_지시",
       semantic_view_refs: [".../SV_예금잔액"] }

[4단계] 에이전트 실행 시 Instruction 해석
  └─ resolve_instruction("financial-advisor", "hong@sh-bank.com", "IT팀")
     → 사용자 배정 확인 → VIP_상담_지시 반환

[5단계] 팀에 공유 (승격)
  └─ promote("VIP_상담_지시", "dept:IT팀", "공유_지시")
     → IT팀 전원이 사용 가능

[6단계] 전사 표준 배포 (승격)
  └─ promote("VIP_상담_지시", "_global", "표준_상담")
     → 전사 에이전트 기본 Instruction으로 사용 가능
```

### UI 네비게이션 구조

```
┌─ 리소스 선택 ──────────────────────────────────┐
│  [📋 Instructions]    [📊 Semantic Views]       │
├─ Scope 선택 ───────────────────────────────────┤
│  [👤 내 뷰]    [👥 IT팀]    [🌐 글로벌]        │
├─ Workspace 선택 ───────────────────────────────┤
│  📁 상담_커스텀 (2)                              │
│  📁 내_실험 (3)                                  │
├─ Items ────────────────────────────────────────┤
│  ◆ VIP_상담_지시          [에이전트 배정 ▶]     │
│  ◆ 테스트_지시_v1                               │
├─ 에이전트 배정 ────────────────────────────────┤
│  🤖 financial-advisor                           │
│     Instruction: VIP_상담_지시 ✅               │
│     Views: SV_예금잔액, SV_예금추이              │
│  🤖 loan-consultant                             │
│     Instruction: (글로벌 기본값)                 │
│     Views: SV_대출잔액                           │
└────────────────────────────────────────────────┘
```

---

## 10. 인덱스 설계

Firestore에서 `collection_group` 쿼리를 사용하려면 **복합 인덱스**를 사전에 생성해야 합니다.

| 컬렉션 그룹 | 인덱스 필드 | 용도 |
|-------------|------------|------|
| `items` | `owner_id` ASC, `updated_at` DESC | 특정 사용자의 전체 Instruction 최신순 |
| `items` | `tags` ARRAY, `scope` ASC | 태그+scope 필터링 |
| `views` | `owner_id` ASC, `updated_at` DESC | 특정 사용자의 전체 Semantic View 최신순 |
| `views` | `tags` ARRAY, `scope` ASC | 태그+scope 필터링 |
| `agents` | `instruction_ref` ASC | 특정 Instruction을 사용 중인 에이전트 역추적 |

```json
// firestore.indexes.json
{
  "indexes": [
    {
      "collectionGroup": "items",
      "queryScope": "COLLECTION_GROUP",
      "fields": [
        { "fieldPath": "owner_id", "order": "ASCENDING" },
        { "fieldPath": "updated_at", "order": "DESCENDING" }
      ]
    },
    {
      "collectionGroup": "views",
      "queryScope": "COLLECTION_GROUP",
      "fields": [
        { "fieldPath": "owner_id", "order": "ASCENDING" },
        { "fieldPath": "updated_at", "order": "DESCENDING" }
      ]
    }
  ]
}
```

---

## 11. 설계 판단 요약

### 단일 컬렉션 vs 계층 컬렉션

| 기준 | 단일 컬렉션 + scope 필드 | 계층 컬렉션 (채택) |
|------|-------------------------|-------------------|
| 쿼리 효율 | 매번 `where` 필터 필요 | 경로만으로 scope 분리 |
| 보안 규칙 | 문서 필드 기반 복잡한 조건 | 경로 기반으로 간결 |
| 전체 검색 | 바로 가능 | `collection_group` 사용 |
| 데이터 격리 | 앱 로직에 의존 | 구조적으로 격리 |
| 확장성 | 문서 수 폭증 시 성능 저하 | scope별 자연 분산 |

### Instruction 참조 방식: 복사 vs 참조

| 방식 | 장점 | 단점 |
|------|------|------|
| **참조 (Firestore 경로)** | 원본 수정 시 자동 반영, 저장 공간 절약 | 원본 삭제 시 깨짐, 읽기 시 추가 조회 |
| **복사 (승격 시)** | 독립적 버전 관리, 원본 변경에 영향 없음 | 동기화 불가, 저장 공간 증가 |

이 설계에서는 **에이전트 배정은 참조**, **승격은 복사** 방식을 혼합합니다. 에이전트가 실행 시점에 항상 최신 Instruction을 사용하되, 승격된 리소스는 독립적으로 관리되어 원본 변경에 영향받지 않도록 합니다.

---

## 정리

Firestore의 계층적 컬렉션 구조를 활용하면, AI Agent의 Instruction과 Semantic View를 **사용자 → 부서 → 글로벌** 3단계로 자연스럽게 관리할 수 있습니다.

핵심 설계 원칙을 다시 정리하면 다음과 같습니다.

| 원칙 | 구현 |
|------|------|
| 경로가 곧 권한 | `_global`, `dept:{id}`, `user:{email}`로 scope 식별 |
| 워크스페이스로 묶기 | 관련 리소스를 논리적 세트로 관리 |
| 참조로 배정 | 에이전트에 Firestore 경로로 Instruction/View 연결 |
| 복사로 승격 | 상위 scope로 올릴 때는 독립 복사본 생성 |
| 우선순위 체인 | 사용자 → 부서 → 글로벌 순으로 폴백 |
| collection_group으로 전체 검색 | scope를 넘나드는 검색은 컬렉션 그룹 쿼리로 해결 |

이 구조는 Firestore의 100단계 중첩 제한 내에서 충분히 동작하며(최대 6단계), 보안 규칙도 경로 패턴만으로 간결하게 작성할 수 있어 운영 부담을 최소화합니다.
