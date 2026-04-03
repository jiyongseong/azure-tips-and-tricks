# API Management에 Microsoft Foundry API 가져오기 가이드

> **목적**: 기존 APIM 인스턴스(`apim-jyseong-demo`)에 **Microsoft Foundry API**를 추가로 가져와서, 이미 등록된 OpenAPI 사양 기반 API와 공존하는 환경을 구성합니다. Foundry API 가져오기 마법사를 사용하면 Operations, 백엔드, Managed Identity 인증, AI 게이트웨이 정책이 **자동 구성**됩니다.
>
> **사전 조건**: [APIM을 통한 Azure OpenAI API 연동 가이드](./APIM을_통한_Azure_OpenAI_API_가이드.md)의 1단계(APIM 생성) 및 2단계(Azure OpenAI 리소스/모델 배포)가 완료되어 있어야 합니다.

---

## 목차

- [1. 사전 환경 요약](#1-사전-환경-요약)
- [2. Microsoft Foundry API 가져오기](#2-microsoft-foundry-api-가져오기)
  - [2.1 클라이언트 호환성 옵션 이해](#21-클라이언트-호환성-옵션-이해)
  - [2.2 Foundry API 가져오기 (포털)](#22-foundry-api-가져오기-포털)
- [3. 자동 구성 항목 확인](#3-자동-구성-항목-확인)
  - [3.1 백엔드 설정 확인](#31-백엔드-설정-확인)
  - [3.2 Managed Identity 및 역할 할당 확인](#32-managed-identity-및-역할-할당-확인)
  - [3.3 Inbound 정책 확인](#33-inbound-정책-확인)
- [4. 포털에서 API 테스트](#4-포털에서-api-테스트)
- [5. Python을 이용한 테스트](#5-python을-이용한-테스트)
  - [5.1 환경 구성](#51-환경-구성)
  - [5.2 APIM 게이트웨이 정보 확인](#52-apim-게이트웨이-정보-확인)
  - [5.3 테스트 코드](#53-테스트-코드)
- [6. 기존 OpenAPI 사양 API와의 공존 확인](#6-기존-openapi-사양-api와의-공존-확인)
- [7. 참고 자료](#7-참고-자료)

---

## 1. 사전 환경 요약

이미 구성되어 있는 리소스 정보입니다. 이 가이드에서는 아래 리소스를 그대로 활용합니다.

| 항목 | 값 |
|------|-----|
| **APIM 인스턴스** | `apim-jyseong-demo` |
| **APIM Gateway URL** | `https://apim-jyseong-demo.azure-api.net` |
| **리소스 그룹** | `rg-foundry-krc` |
| **Azure OpenAI 리소스** | `openai-jyseong-demo` |
| **모델 배포** | `gpt-4.1` |
| **기존 API (OpenAPI 사양)** | `Azure OpenAI API` (API URL suffix: `openai-api`) |

> 이 가이드를 완료하면 APIM에 아래 두 API가 공존하게 됩니다:
>
> | API | 등록 방식 | API URL suffix (예시) |
> |-----|----------|----------------------|
> | Azure OpenAI API | OpenAPI 사양 파일 가져오기 | `openai-api` |
> | **Foundry OpenAI API** | **Microsoft Foundry 가져오기** | `foundry-api` |

---

## 2. Microsoft Foundry API 가져오기

### 2.1 클라이언트 호환성 옵션 이해

Foundry API를 가져올 때 **Client compatibility** 옵션을 선택해야 합니다. 배포된 모델 유형에 따라 적합한 옵션이 다릅니다.

| 옵션 | 엔드포인트 형식 | 배포 이름 위치 | 적합한 경우 |
|------|----------------|---------------|------------|
| **Azure OpenAI** | `/openai/deployments/{deployment}/chat/completions` | URL 경로 | Azure OpenAI 모델만 사용할 때 |
| **Azure AI** | `/models/chat/completions` | 요청 body | Azure AI Model Inference API를 사용하는 다양한 모델을 관리할 때 |
| **Azure OpenAI v1** | `/openai/v1/chat/completions` | 요청 body (`model`) | Azure OpenAI API v1 형식을 사용할 때 (현재 환경에 해당) |

> **권장**: **Azure OpenAI v1** 옵션을 선택합니다. 요청 body의 `model` 필드로 배포 이름을 지정하는 방식으로, URL 경로에 배포 이름을 포함하지 않아 유연합니다.

### 2.2 Foundry API 가져오기 (포털)

1. [Azure Portal](https://portal.azure.com/)에 로그인 → `apim-jyseong-demo` 인스턴스로 이동합니다.

2. 왼쪽 메뉴 **APIs** → **APIs** → **+ Add API**를 클릭합니다.

3. **Create from Azure resource** 섹션에서 **Microsoft Foundry**를 선택합니다.

4. **Select AI Service** 탭:

   a. **Subscription**: 현재 구독을 선택합니다.

   b. Foundry 서비스 목록에서 **`openai-jyseong-demo`**를 선택합니다.
      - 서비스 이름 옆의 **deployments** 링크를 클릭하면 해당 서비스에 배포된 모델 목록을 확인할 수 있습니다.

   c. **Next**를 클릭합니다.

5. **Configure API** 탭:

   | 항목 | 설명 | 예시 값 |
   |------|------|---------|
   | **Display name** | APIM에 표시될 API 이름 | `Foundry OpenAI API` |
   | **Description** | API 설명 (선택) | `Microsoft Foundry를 통한 OpenAI API` |
   | **Base path** | API URL suffix (기존 API와 **중복 불가**) | `foundry-api` |
   | **Products** | 연결할 Product (선택) | — |
   | **Client compatibility** | 클라이언트 호환성 옵션 | **Azure OpenAI v1** |

   > ⚠️ **중요**: Base path(API URL suffix)는 기존 `openai-api`와 **다른 값**을 사용해야 합니다. 같은 값을 사용하면 충돌이 발생합니다.

   c. **Next**를 클릭합니다.

6. **Manage token consumption** 탭:

   토큰 소비 관리 정책을 설정합니다. 기본값을 유지하거나 필요에 따라 조정합니다.
   - **Token limit**: TPM(Tokens Per Minute) 제한 설정
   - **Token usage tracking**: 토큰 사용량 추적 메트릭 활성화

   > 💡 테스트 목적이라면 기본값을 유지하고 **Next**를 클릭합니다.

7. **Apply semantic caching** 탭:

   의미 기반 캐싱 정책을 설정합니다. 유사한 프롬프트에 대해 캐시된 응답을 반환하여 비용과 지연을 줄입니다.

   > 💡 테스트 목적이라면 기본값을 유지하고 **Next**를 클릭합니다.

8. **AI content safety** 탭:

   Azure AI Content Safety 서비스를 연결하여 안전하지 않은 콘텐츠를 차단합니다.

   > 💡 테스트 목적이라면 기본값을 유지하고 **Next**를 클릭합니다.

9. **Review** 단계에서 설정을 검토한 후 **Create**를 클릭합니다.

> ✅ **자동 구성**: 마법사가 완료되면 아래 항목이 자동으로 생성/설정됩니다:
> - REST API 엔드포인트별 **Operations**
> - **백엔드** 리소스 (Azure AI Services 엔드포인트)
> - **`set-backend-service`** Inbound 정책
> - **시스템 할당 Managed Identity**를 사용한 백엔드 인증
> - APIM → OpenAI 리소스에 대한 **Cognitive Services OpenAI User** 역할 할당

---

## 3. 자동 구성 항목 확인

Foundry API 가져오기가 완료되면 자동으로 구성된 항목들을 확인합니다.

### 3.1 백엔드 설정 확인

1. APIM 인스턴스 → 왼쪽 메뉴 **APIs** → **Backends**를 선택합니다.

2. Foundry 가져오기로 생성된 새 백엔드가 추가되어 있는지 확인합니다.
   - **Runtime URL**: `https://openai-jyseong-demo.openai.azure.com/openai` 형태

3. 백엔드를 클릭 → **Authorization credentials** → **Managed Identity** 탭에서:
   - **Enable**: 체크됨
   - **Resource ID**: `https://cognitiveservices.azure.com`

   > 이전에 수동으로 구성했던 것과 달리, Foundry 가져오기는 이 설정을 **자동으로** 처리합니다.

### 3.2 Managed Identity 및 역할 할당 확인

1. APIM 인스턴스 → 왼쪽 메뉴 **Security** → **Managed identities**를 확인합니다.
   - **System assigned** 탭의 **Status**가 **On**인지 확인합니다.

2. `openai-jyseong-demo` 리소스 → **Access control (IAM)** → **Role assignments** 탭에서:
   - `apim-jyseong-demo`에 **Cognitive Services OpenAI User** 역할이 할당되어 있는지 확인합니다.

   > 가져오기 마법사에서 이미 자동 할당되었을 것입니다.

### 3.3 Inbound 정책 확인

1. APIM → **APIs** → `Foundry OpenAI API` → **All operations** → **Policies** (코드 편집기)를 확인합니다.

2. 자동 생성된 정책은 대략 다음과 같은 구조입니다:

```xml
<policies>
    <inbound>
        <base />
        <set-backend-service backend-id="foundry-openai-backend" />
        <!-- 추가 AI 게이트웨이 정책 (Token limit, Emit token metric 등) -->
    </inbound>
    <backend>
        <base />
    </backend>
    <outbound>
        <base />
    </outbound>
    <on-error>
        <base />
    </on-error>
</policies>
```

> 💡 Foundry 가져오기에서는 `authentication-managed-identity`나 `set-header api-key`를 Inbound 정책에 명시하지 않아도 됩니다. 백엔드의 **Authorization credentials** 설정에서 Managed Identity 인증이 자동으로 적용됩니다.

---

## 4. 포털에서 API 테스트

Python 코드를 실행하기 전에, 먼저 APIM 포털의 테스트 콘솔에서 API가 정상 작동하는지 확인합니다.

1. APIM → **APIs** → `Foundry OpenAI API`를 선택합니다.

2. **Test** 탭을 클릭합니다.

3. **Creates a chat completion** (또는 유사한 Chat Completions) Operation을 선택합니다.

4. **Request body**에 다음을 입력합니다:

   > ⚠️ Azure OpenAI v1 형식에서는 **`model` 필드에 배포 이름을 반드시 포함**해야 합니다. 누락하면 `400 Bad Request` — `"Missed model deployment"` 오류가 발생합니다.

```json
{
    "model": "gpt-4.1",
    "messages": [
        {
            "role": "user",
            "content": "안녕하세요, 테스트입니다."
        }
    ],
    "max_tokens": 100
}
```

> 💡 테스트 콘솔에서는 **Ocp-Apim-Subscription-Key** 헤더가 자동으로 설정됩니다.

5. **Send**를 클릭합니다.

6. **HTTP response** 섹션에서 `200 OK` 응답과 함께 모델의 답변이 반환되는지 확인합니다.

---

## 5. Python을 이용한 테스트

### 5.1 환경 구성

```bash
pip install openai
```

### 5.2 APIM 게이트웨이 정보 확인

1. Azure Portal → **APIM 인스턴스** → **개요(Overview)** 페이지

2. **Gateway URL** 값을 확인합니다.
   - 예: `https://apim-jyseong-demo.azure-api.net`

3. API URL suffix를 결합하면 최종 엔드포인트가 됩니다:
   - **최종 엔드포인트**: `https://apim-jyseong-demo.azure-api.net/foundry-api`
   - > Azure OpenAI v1 형식에서는 실제 호출 URL이 `/foundry-api/openai/v1/chat/completions`이 됩니다. 배포 이름은 Request body의 `model` 필드로 지정합니다.

#### Subscription Key 확인

> ⚠️ **Foundry API 가져오기의 Subscription 헤더 차이**
>
> Foundry API 가져오기로 등록된 API는 Subscription Key 헤더 이름이 기본값(`Ocp-Apim-Subscription-Key`)이 아닌 **`api-key`**로 자동 설정됩니다.
>
> | 항목 | OpenAPI 사양 가져오기 | Foundry 가져오기 |
> |------|---------------------|----------------|
> | **Header name** | `Ocp-Apim-Subscription-Key` | `api-key` |
> | **Query parameter** | `subscription-key` | `subscription-key` |
>
> 이 설정은 APIM → APIs → 해당 API → **Settings** 탭 → **Subscription** 섹션에서 확인할 수 있습니다.
> **주의**: `AzureOpenAI` 클라이언트는 `api_key`를 `api-key` 헤더로 전송하지만, `OpenAI`(표준) 클라이언트는 `Authorization: Bearer` 헤더로 전송합니다. v1 형식에서는 `OpenAI` 클라이언트를 사용하므로, `default_headers={"api-key": APIM_SUBSCRIPTION_KEY}`로 명시적으로 헤더를 설정해야 합니다.

1. APIM 인스턴스 → 왼쪽 메뉴 **APIs** → **Subscriptions**를 선택합니다.

2. **Built-in all-access subscription** 행의 오른쪽 **...** 메뉴 → **Show/hide keys**를 클릭합니다.

3. **Primary key** 값을 복사합니다.

### 5.3 테스트 코드

아래 코드를 `test_foundry_via_apim.py` 파일로 저장하고 실행합니다.

```python
import os
from openai import OpenAI

# ============================================================
# 환경 변수 또는 직접 입력
# ============================================================
# APIM Gateway URL + Foundry API URL suffix
# 예: https://apim-jyseong-demo.azure-api.net/foundry-api
APIM_ENDPOINT = os.environ.get("APIM_ENDPOINT", "https://<apim-name>.azure-api.net/<api-url-suffix>")
BASE_URL = f"{APIM_ENDPOINT}/openai/v1"

# APIM Subscription Key (Subscriptions 메뉴에서 확인)
APIM_SUBSCRIPTION_KEY = os.environ.get("APIM_SUBSCRIPTION_KEY", "<your-apim-subscription-key>")

# Azure OpenAI 배포 이름 (예: gpt-4.1)
DEPLOYMENT = os.environ.get("DEPLOYMENT", "gpt-4.1")

# ============================================================
# Azure OpenAI v1 형식은 api-version 쿼리 파라미터가 필요 없습니다.
# OpenAI 클라이언트를 사용하여 base_url에 v1 엔드포인트를 지정합니다.
#
# Foundry 가져오기로 등록된 API는 Subscription 헤더 이름이 "api-key"입니다.
# OpenAI(표준) 클라이언트는 api_key를 "Authorization: Bearer" 헤더로 전송하므로,
# default_headers로 "api-key" 헤더를 직접 설정해야 합니다.
# ============================================================

client = OpenAI(
    base_url=BASE_URL,
    api_key="placeholder",  # APIM 백엔드가 Managed Identity로 인증하므로 실제 OpenAI 키 불필요
    default_headers={"api-key": APIM_SUBSCRIPTION_KEY},  # Foundry API Subscription 인증
)

print(f"Base URL: {BASE_URL}")
print(f"Model:    {DEPLOYMENT}")
print("-" * 60)

response = client.chat.completions.create(
    messages=[
        {
            "role": "system",
            "content": "You are a helpful assistant.",
        },
        {
            "role": "user",
            "content": "안녕 너는 누구니",
        },
    ],
    max_completion_tokens=13107,
    temperature=1.0,
    top_p=1.0,
    frequency_penalty=0.0,
    presence_penalty=0.0,
    model=DEPLOYMENT,
)

# 응답 출력
print("=" * 60)
print("응답 내용:")
print(response.choices[0].message.content)
print("=" * 60)
print(f"모델: {response.model}")
print(f"사용 토큰 - 프롬프트: {response.usage.prompt_tokens}, 완료: {response.usage.completion_tokens}, 총: {response.usage.total_tokens}")
```

#### 실행 방법

**Windows PowerShell**:
```powershell
$env:APIM_ENDPOINT = "https://apim-jyseong-demo.azure-api.net/foundry-api"
$env:APIM_SUBSCRIPTION_KEY = "your-subscription-key-here"
$env:DEPLOYMENT = "gpt-4.1"

python test_foundry_via_apim.py
```

**macOS / Linux (Bash)**:
```bash
export APIM_ENDPOINT="https://apim-jyseong-demo.azure-api.net/foundry-api"
export APIM_SUBSCRIPTION_KEY="your-subscription-key-here"
export DEPLOYMENT="gpt-4.1"

python test_foundry_via_apim.py
```

#### 예상 출력

```
Base URL: https://apim-jyseong-demo.azure-api.net/foundry-api/openai/v1
Model:    gpt-4.1
------------------------------------------------------------
============================================================
응답 내용:
안녕하세요! 저는 AI 어시스턴트입니다. 질문에 답하거나 정보를 제공하는 등 다양한 방법으로 도움을 드릴 수 있습니다. 무엇을 도와드릴까요?
============================================================
모델: gpt-4.1
사용 토큰 - 프롬프트: 24, 완료: 52, 총: 76
```

---

## 6. 기존 OpenAPI 사양 API와의 공존 확인

두 API가 APIM에서 정상적으로 공존하는지 확인합니다.

### API 목록 확인

APIM → **APIs** → 왼쪽 API 목록에서 다음 두 API가 모두 나타나야 합니다:

| API 이름 | 등록 방식 | API URL suffix | 백엔드 인증 |
|---------|----------|---------------|------------|
| Azure OpenAI API | OpenAPI 사양 파일 가져오기 | `openai-api` | Managed Identity (수동 구성) |
| Foundry OpenAI API | Microsoft Foundry 가져오기 | `foundry-api` | Managed Identity (자동 구성) |

### 엔드포인트 비교

두 API 모두 같은 APIM 게이트웨이를 통해 접근하지만, URL suffix로 구분됩니다:

```
# OpenAPI 사양 기반 API
https://apim-jyseong-demo.azure-api.net/openai-api/openai/deployments/gpt-4.1/chat/completions?api-version=2024-12-01-preview

# Foundry 기반 API (Azure OpenAI v1)
https://apim-jyseong-demo.azure-api.net/foundry-api/openai/v1/chat/completions
# → Request body에 "model": "gpt-4.1" 포함
```

### 차이점 요약

| 비교 항목 | OpenAPI 사양 가져오기 | Foundry 가져오기 |
|----------|---------------------|----------------|
| **등록 방법** | JSON/YAML 사양 파일 업로드 | 포털 마법사에서 Foundry 서비스 선택 |
| **Operations 자동 생성** | 사양 파일 의존 (외부 `$ref` 이슈 가능) | 자동 생성 |
| **백엔드 구성** | 수동 설정 필요 | 자동 생성 |
| **Managed Identity 인증** | 수동 구성 (정책 또는 백엔드 설정) | 자동 구성 |
| **IAM 역할 할당** | 수동 할당 필요 | 자동 할당 |
| **AI 게이트웨이 정책** | 수동 추가 필요 | 마법사에서 선택적 구성 |
| **적합한 시나리오** | 세밀한 API 정의 제어가 필요할 때 | 빠르게 시작하고 자동 구성을 활용할 때 |
| **Subscription 헤더** | `Ocp-Apim-Subscription-Key` | `api-key` |
---

## 7. 참고 자료

| 항목 | 링크 |
|------|------|
| Microsoft Foundry API 가져오기 | https://learn.microsoft.com/en-us/azure/api-management/azure-ai-foundry-api |
| APIM AI 게이트웨이 기능 | https://learn.microsoft.com/en-us/azure/api-management/genai-gateway-capabilities |
| LLM 토큰 제한 정책 | https://learn.microsoft.com/en-us/azure/api-management/llm-token-limit-policy |
| LLM 토큰 메트릭 정책 | https://learn.microsoft.com/en-us/azure/api-management/llm-emit-token-metric-policy |
| 시맨틱 캐싱 활성화 | https://learn.microsoft.com/en-us/azure/api-management/azure-openai-enable-semantic-caching |
| AI API 인증/권한 부여 | https://learn.microsoft.com/en-us/azure/api-management/api-management-authenticate-authorize-ai-apis |
| Azure OpenAI REST API 사양 | https://learn.microsoft.com/en-us/azure/ai-services/openai/reference |
| OpenAI Python SDK | https://github.com/openai/openai-python |

✍️ 2026년 4월 2일 씀.