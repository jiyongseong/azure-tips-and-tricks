# Microsoft Foundry 리소스 및 프로젝트 구조 가이드

## 목차

1. [개요](#1-개요)
2. [Foundry 리소스 단위 설정 가능 옵션](#2-foundry-리소스-단위-설정-가능-옵션)
3. [Project 단위 설정 가능 옵션](#3-project-단위-설정-가능-옵션)
4. [고객 시나리오 분석](#4-고객-시나리오-분석)
5. [Best Practice 권고안](#5-best-practice-권고안)
6. [참고 문서](#6-참고-문서)

---

## 1. 개요

### 1.1 Foundry 리소스와 프로젝트의 관계

Microsoft Foundry는 **Foundry 리소스(Resource)** 와 **Foundry 프로젝트(Project)** 의 Parent-Child 구조로 구성된다.

```
Azure Subscription
  └─ Resource Group
       └─ Foundry 리소스 (Microsoft.CognitiveServices/account, kind=AIServices)
            ├─ Project A (Microsoft.CognitiveServices/account/project)
            ├─ Project B
            └─ Project C
```

| 구분 | Azure 리소스 유형 | 역할 |
|------|-------------------|------|
| **Foundry 리소스** | `Microsoft.CognitiveServices/account` (kind: AIServices) | 관리, 보안, 모니터링의 **최상위 경계**. 네트워크, 암호화, 모델 배포 등 공유 설정을 관리 |
| **Foundry 프로젝트** | `Microsoft.CognitiveServices/account/project` | Foundry 리소스 내의 **개발 경계**. 에이전트, 평가, 파일, 인덱스 등 작업을 조직하고 접근 제어를 적용 |

> **핵심 원칙**: IT 팀은 Foundry 리소스 수준에서 중앙 집중식 제어를 적용하고, 개발 팀은 프로젝트 수준의 경계 내에서 작업한다.

### 1.2 Default Project vs Non-Default Project

Foundry 리소스의 첫 번째 프로젝트는 "default" 프로젝트로 지정된다. Default 프로젝트와 Non-Default 프로젝트 간에는 기능 차이가 존재한다.

| 기능 | Default Project | Non-Default Project |
|------|:-:|:-:|
| Model inference, Playgrounds, Agents | ✅ | ✅ |
| Evaluations, Tracing, Datasets, Indexes | ✅ | ✅ |
| Foundry SDK and API | ✅ | ✅ |
| Content Understanding | ✅ | ✅ |
| OpenAI SDK and API | ✅ | Responses, Files, Conversations |
| **OpenAI Batch, Fine-tuning, Stored completions** | ✅ | ❌ |
| **Speech Fine-tuning** | ✅ | ❌ |
| Language Fine-tuning | ✅ | ✅ |
| Connections | ✅ | ✅ |

> **참고**: Default 프로젝트를 삭제하면, 다음에 생성되는 프로젝트가 자동으로 Default 프로젝트가 된다.

---

## 2. Foundry 리소스 단위 설정 가능 옵션

Foundry 리소스는 **관리(Control Plane)** 의 경계이며, 하위 모든 프로젝트에 공유되는 설정을 관리한다.

| # | 설정 항목 | 설명 | 비고 |
|---|----------|------|------|
| 1 | **RBAC (리소스 범위)** | `Azure AI Account Owner`, `Azure AI Project Manager`, `Azure AI Owner` 등의 역할을 리소스 범위에서 할당. 리소스 내 모든 프로젝트에 권한이 상속됨 | 리소스 범위 할당 시 하위 프로젝트 전체에 영향 |
| 2 | **네트워크 보안** | Private Endpoint, VNet 격리, Azure Firewall 구성. 모든 프로젝트가 동일한 네트워크 설정을 공유 | 활성화 후 비활성화 불가 |
| 3 | **모델 배포 관리** | LLM/SLM 모델 배포(Deployment) 및 TPM(Tokens Per Minute) 제한 설정. 모든 프로젝트가 공유 | Standard/Provisioned 배포 유형 선택 |
| 4 | **공유 연결 (Connections)** | Azure Storage, Key Vault 등에 대한 리소스 수준 연결. 모든 프로젝트에서 접근 가능 | 공유 인증 토큰(Managed Identity 또는 API 키) 사용 |
| 5 | **암호화 (CMK)** | Customer-Managed Key 또는 Microsoft-Managed Key 설정. 리소스 내 전체 데이터에 적용 | Key Vault는 동일 리전에 배포 필요 |
| 6 | **Azure Key Vault 연결** | API 키 기반 연결 시크릿을 자체 Key Vault로 관리 가능 | 기본값은 Microsoft 관리형 Key Vault |
| 7 | **Azure Policy** | 모델 접근 제한, 리소스 생성 규칙 강제 등 거버넌스 정책 적용 | 구독/리소스 그룹 수준에서도 적용 가능 |
| 8 | **진단 로그 / 모니터링** | Azure Monitor, Log Analytics, 진단 설정(Audit, RequestResponse, AllMetrics) | 리소스 단위 중앙 집중식 로그 관리 |
| 9 | **비용 관리** | Microsoft Cost Management를 통한 비용 추적 단위. 리소스별/리소스 그룹별 비용 분석 가능 | 비즈니스 그룹별 비용 추적에 핵심 |
| 10 | **Managed Identity (시스템 할당)** | Foundry 리소스의 시스템 할당 관리 ID. 다른 Azure 서비스 접근 시 사용 | 프로젝트 수준 Managed Identity와 별도 |
| 11 | **Quota 관리** | 모델별/리전별 할당량 관리 (TPM 기반) | 리소스 단위로 할당량이 적용됨 |
| 12 | **태그 (Tags)** | Azure 리소스 태그를 통한 소유자, 환경, 비용 센터 추적 | 거버넌스 및 비용 추적에 활용 |
| 13 | **인증 방법 설정** | API 키 인증 활성화/비활성화, Microsoft Entra ID 인증 | 프로덕션에서는 키 인증 비활성화 권장 |

---

## 3. Project 단위 설정 가능 옵션

프로젝트는 **개발(Data Plane)** 의 경계이며, 개별 Use Case 단위로 작업을 격리한다.

| # | 설정 항목 | 설명 | 비고 |
|---|----------|------|------|
| 1 | **RBAC (프로젝트 범위)** | `Azure AI User` 역할을 프로젝트 범위에서 할당하여 특정 프로젝트에만 접근 허용 | 완전 격리 모델에서 핵심 |
| 2 | **프로젝트 범위 Connections** | 민감한 데이터 소스에 대한 프로젝트 전용 연결. Microsoft Entra ID Passthrough 인증 지원 | 리소스 수준 공유 연결과 별도 |
| 3 | **프로젝트 Managed Identity** | 프로젝트별 시스템 할당 관리 ID. Foundry 리소스에서 `Azure AI User` 역할 할당 필요 | 프로젝트 생성 시 자동 생성 |
| 4 | **자산 격리** | 에이전트(Agents), 평가(Evaluations), 파일(Files), 인덱스(Indexes), 데이터셋(Datasets) | 프로젝트 간 자산은 격리됨 |
| 5 | **Application Insights 연결** | 프로젝트별 Tracing 및 성능 모니터링 | `Azure Monitor Reader` 역할 추가 필요 |
| 6 | **프로젝트별 메트릭** | 평가 성능, 에이전트 활동 등 프로젝트 범위 메트릭 | Azure Monitor에서 프로젝트별 조회 가능 |

---

## 4. 시나리오 분석

### 4.1 패턴 A: 프로젝트(팀/개발그룹) 단위로 Foundry 리소스 생성 후 prd/dev 프로젝트 분리

```
1번 Foundry (개발 1번 프로젝트)
   ├─ 1번 pjt - prd
   └─ 1번 pjt - dev
2번 Foundry (개발 2번 프로젝트)
   ├─ 2번 pjt - prd
   └─ 2번 pjt - dev
```

| 항목 | 평가 |
|------|------|
| **장점** | 프로젝트(팀) 단위의 강력한 격리. 네트워크/비용/모델 배포가 완전 분리됨 |
| **단점** | ⚠️ **prd/dev가 동일 Foundry 리소스 내에 존재**하여 네트워크 보안, 모델 배포, 연결 설정이 공유됨. 운영 환경과 개발 환경의 보안 수준이 동일하게 적용되어 유연성이 떨어짐. Foundry 리소스가 많아져 관리 부담 증가 |
| **Microsoft 권고 부합 여부** | ❌ 환경(prd/dev) 분리 원칙에 부합하지 않음. Microsoft는 **dev/test/prod를 별도 Foundry 리소스(또는 구독/리소스 그룹)로 분리**할 것을 권고 |

### 4.2 패턴 B: 소수 Foundry에 다수 프로젝트(prd/dev 포함) 집중 배치

```
1번 Foundry
   ├─ 1번 pjt - prd
   ├─ 1번 pjt - dev
   ├─ 2번 pjt - prd
   ├─ 2번 pjt - dev
   ├─ 3번 pjt - prd
   ├─ 3번 pjt - dev
   └─ ...
2번 Foundry
   ├─ 101번 pjt - prd
   ├─ 101번 pjt - dev
   └─ ...
```

| 항목 | 평가 |
|------|------|
| **장점** | Foundry 리소스 수가 적어 관리 간소화. 모델 배포와 연결을 많은 프로젝트가 공유하여 효율적 |
| **단점** | ⚠️ **prd/dev가 동일 리소스에 혼재**되어 보안 경계 모호. 한 리소스에 과도한 프로젝트 집중 시 Quota/성능 경합 가능. RBAC만으로 prd/dev 간 완전 격리를 보장하기 어려움 (네트워크, 모델 배포는 공유). 비용 추적이 비즈니스 그룹 단위로 되지 않음 |
| **Microsoft 권고 부합 여부** | ❌ 비즈니스 그룹별 분리 원칙과 환경 분리 원칙 모두에 부합하지 않음 |

### 4.3 패턴 C (권고): 비즈니스 그룹 × 환경 단위로 Foundry 리소스 분리

```
개발 1팀 - Dev Foundry (RG: rg-team1-dev)
   ├─ Use Case A (Project)
   ├─ Use Case B (Project)
   └─ Use Case C (Project)

개발 1팀 - Prd Foundry (RG: rg-team1-prd)
   ├─ Use Case A (Project)
   ├─ Use Case B (Project)
   └─ Use Case C (Project)

개발 2팀 - Dev Foundry (RG: rg-team2-dev)
   ├─ Use Case D (Project)
   └─ Use Case E (Project)

개발 2팀 - Prd Foundry (RG: rg-team2-prd)
   ├─ Use Case D (Project)
   └─ Use Case E (Project)
```

| 항목 | 평가 |
|------|------|
| **장점** | ✅ 비즈니스 그룹별 자율성과 거버넌스 보장. ✅ prd/dev 환경이 완전히 분리되어 네트워크/보안/모델 배포 독립 관리. ✅ 비용이 비즈니스 그룹 × 환경 단위로 명확히 추적됨. ✅ 프로젝트는 Use Case 단위로 구성되어 자산 격리와 팀 협업 최적화 |
| **단점** | Foundry 리소스 수가 비교적 많아질 수 있음. IaC(Bicep/Terraform)를 통한 자동화 필요 |
| **Microsoft 권고 부합 여부** | ✅ **완전 부합**. Microsoft의 3가지 핵심 원칙 모두 충족 |

---

## 5. Best Practice 권고안

### 5.1 Microsoft 공식 권고 핵심 원칙

Microsoft의 [Foundry Rollout 가이드](https://learn.microsoft.com/en-us/azure/foundry/concepts/planning)에서 명시한 3가지 핵심 원칙:

| # | 원칙 | 원문 |
|---|------|------|
| 1 | **비즈니스 그룹별 별도 Foundry 리소스 생성** | *"Create a separate Foundry resource for each business group. Align deployments with logical boundaries such as data domains or business functions to ensure autonomy, governance, and cost tracking."* |
| 2 | **dev/test/prod 환경별 별도 Foundry 리소스 분리** | *"Establish distinct environments for development, testing, and production. Use separate resource groups or subscriptions, and Foundry resources to isolate workflows, manage access, and support experimentation with controlled releases."* |
| 3 | **프로젝트는 Use Case 단위로 구성** | *"Associate projects with use cases. Foundry projects are designed to represent specific use cases. They're containers to organize components such as agents or files for an application."* |

### 5.2 Foundry 리소스 분리 기준 체크리스트

다음 항목 중 **하나라도 다른 경우** Foundry 리소스를 분리하는 것이 권장된다:

| 분리 기준 | 설명 | 예시 |
|-----------|------|------|
| **보안 경계** | 네트워크 격리 수준이 다른 경우 | 인터넷 허용 (dev) vs 완전 격리 (prd) |
| **환경 분리** | dev/test/prod 환경 간 워크플로 격리 | 개발 환경에서 실험적 모델 사용, 운영 환경에서 검증된 모델만 사용 |
| **비즈니스 그룹** | 소유권, 자율성, 비용 책임이 다른 조직 | 개발 1팀 vs 개발 2팀 |
| **규정 준수** | 데이터 주권, 산업 규정 등이 다른 경우 | 금융 데이터 vs 일반 데이터 |
| **비용 추적** | 독립적인 비용 관리가 필요한 경우 | 사업부별 예산 분리 |
| **모델 배포 전략** | 배포 유형(Standard/Provisioned) 또는 모델 버전이 다른 경우 | dev에서 최신 프리뷰 모델 테스트, prd에서 안정 GA 버전 사용 |

### 5.3 프로젝트 분리 기준 체크리스트

Foundry 리소스 내에서 프로젝트를 분리하는 기준:

| 분리 기준 | 설명 | 예시 |
|-----------|------|------|
| **Use Case / 애플리케이션** | 하나의 에이전트 또는 AI 애플리케이션 단위 | 고객 상담 봇, 문서 요약 에이전트, 코드 리뷰 에이전트 |
| **접근 제어** | 프로젝트별로 다른 개발자 그룹이 작업하는 경우 | A팀은 Project A만, B팀은 Project B만 접근 |
| **데이터 격리** | 프로젝트별로 민감도가 다른 데이터를 사용하는 경우 | 공개 데이터 기반 프로젝트 vs 내부 기밀 데이터 프로젝트 |
| **라이프사이클** | 개발/배포/운영 주기가 다른 경우 | 빠르게 실험하는 PoC vs 장기 운영 서비스 |

### 5.4 권고 구조 (엔터프라이즈)

```
Azure Subscription (또는 비즈니스 그룹별 구독)
├─ RG: rg-{비즈니스그룹}-dev
│    └─ Foundry 리소스 (Dev)
│         ├─ Project: use-case-A
│         ├─ Project: use-case-B
│         └─ Project: use-case-C
│
├─ RG: rg-{비즈니스그룹}-prd
│    └─ Foundry 리소스 (Prd)
│         ├─ Project: use-case-A
│         ├─ Project: use-case-B
│         └─ Project: use-case-C
│
├─ RG: rg-{비즈니스그룹2}-dev
│    └─ Foundry 리소스 (Dev)
│         └─ ...
└─ RG: rg-{비즈니스그룹2}-prd
     └─ Foundry 리소스 (Prd)
          └─ ...
```

### 5.5 RBAC 매핑 권고

| 역할 | 할당할 Azure 역할 | 범위 | 설명 |
|------|------------------|------|------|
| IT 관리자 | Owner | 구독 범위 | Foundry 리소스가 기업 표준을 준수하는지 확인 |
| 관리자 | Azure AI Account Owner | Foundry 리소스 범위 | 모델 배포, 컴퓨팅 리소스 감사, 공유 연결 생성 |
| 팀 리드 | Azure AI Project Manager | Foundry 리소스 범위 | 프로젝트 생성, 멤버 초대, Azure AI User 역할 할당 |
| 개발자 | Azure AI User + Reader | 프로젝트 범위 + 리소스 범위 | 할당된 프로젝트 내에서만 에이전트 빌드 |

### 5.6 구현 시 고려사항

1. **IaC(Infrastructure as Code)** 활용: Bicep/Terraform 템플릿으로 Foundry 리소스 및 프로젝트 생성을 자동화하여 일관성 보장
2. **Microsoft Entra ID 그룹**: 개별 사용자 대신 보안 그룹 단위로 역할을 할당하여 관리 효율 극대화
3. **네이밍 규칙 수립**: `{조직}-{비즈니스그룹}-{환경}` 형식으로 리소스 이름을 표준화 (예: `contoso-team1-dev`)
4. **Azure Policy 적용**: 프리뷰 모델 접근 제한, 리소스 생성 위치 제한 등 거버넌스 규칙 강제
5. **태그(Tag) 전략**: 리소스에 `Environment`, `BusinessGroup`, `CostCenter`, `Owner` 태그를 적용
6. **Quota 사전 확인**: 각 리전의 모델별 TPM 할당량을 확인하고, 필요 시 사전에 증량 요청
7. **Default Project 인지**: OpenAI Batch, Fine-tuning 등 일부 기능은 Default 프로젝트에서만 사용 가능하므로, 핵심 워크로드를 Default 프로젝트에 배치할 것을 고려

---

## 6. 참고 문서

### 6.1 Microsoft 공식 문서

| 문서 | URL | 주요 내용 |
|------|-----|----------|
| **Foundry Architecture** | https://learn.microsoft.com/en-us/azure/foundry/concepts/architecture | 리소스-프로젝트 관계, 보안 분리 원칙, 컴퓨팅/스토리지 아키텍처 |
| **Foundry Rollout Planning** | https://learn.microsoft.com/en-us/azure/foundry/concepts/planning | 환경 분리 권고, Contoso 예시, 비즈니스 그룹별 구성, 보안/연결/거버넌스 |
| **RBAC for Foundry** | https://learn.microsoft.com/en-us/azure/foundry/concepts/rbac-foundry | Built-in 역할, Custom 역할, 액세스 격리 옵션, 엔터프라이즈 매핑 예시 |
| **Create Projects** | https://learn.microsoft.com/en-us/azure/foundry/how-to/create-projects | 프로젝트 생성, 다중 프로젝트 구성, Default vs Non-Default 기능 차이 |
| **Private Link 구성** | https://learn.microsoft.com/en-us/azure/foundry/how-to/configure-private-link | 네트워크 격리, Private Endpoint, VNet 통합 |
| **CMK 암호화** | https://learn.microsoft.com/en-us/azure/foundry/concepts/encryption-keys-portal | Customer-Managed Key 설정 및 요구사항 |
| **인증 및 권한 부여** | https://learn.microsoft.com/en-us/azure/foundry/concepts/authentication-authorization-foundry | Entra ID 인증, API 키, 인증 흐름 |
| **Connections 관리** | https://learn.microsoft.com/en-us/azure/foundry/how-to/connections-add | 리소스/프로젝트 수준 연결 구성 |
| **Bicep 템플릿** | https://learn.microsoft.com/en-us/azure/foundry/how-to/create-resource-template | 리소스 자동화 배포 템플릿 |
| **모델 배포 유형** | https://learn.microsoft.com/en-us/azure/ai-services/openai/how-to/deployment-types | Standard vs Provisioned 배포 |
| **Azure Policy (모델 제한)** | https://learn.microsoft.com/en-us/azure/foundry-classic/how-to/built-in-policy-model-deployment | 모델 접근 정책 구성 |
| **리전별 기능 가용성** | https://learn.microsoft.com/en-us/azure/foundry/reference/region-support | 리전별 모델/기능 사용 가능 여부 확인 |

### 6.2 관련 내부 문서

- `Foundry/Governance/Microsoft Foundry와 IAM 기능의 역할.md` — RBAC, 네트워크 보안, 거버넌스, 감사/모니터링 상세 가이드

✍️ 2026년 4월 8일 씀.