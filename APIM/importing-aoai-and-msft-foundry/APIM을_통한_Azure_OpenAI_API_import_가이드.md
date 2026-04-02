# API Management를 통한 Azure OpenAI API 연동 가이드

> **목적**: Azure API Management(APIM)를 통해 Azure OpenAI 모델 엔드포인트를 관리하고, Python 클라이언트에서 호출하는 전체 과정을 단계별로 안내

---

## 목차

- [1. API Management 인스턴스 생성](#1-api-management-인스턴스-생성)
- [2. Azure OpenAI 리소스 및 모델 배포](#2-azure-openai-리소스-및-모델-배포)
  - [2.1 Azure OpenAI 리소스 생성](#21-azure-openai-리소스-생성)
  - [2.2 모델 배포 (gpt-4.1)](#22-모델-배포-gpt-41)
- [3. API Management에 OpenAI API 사양 추가](#3-api-management에-openai-api-사양-추가)
  - [3.1 방법 A — OpenAPI 사양 파일로 등록](#31-방법-a--openapi-사양-파일로-등록)
  - [3.2 방법 B — Language Model API로 등록](#32-방법-b--language-model-api로-등록)
- [4. 인증 구성](#4-인증-구성)
  - [4.1 방법 A — API Key 방식](#41-방법-a--api-key-방식)
  - [4.2 방법 B — Managed Identity 방식](#42-방법-b--managed-identity-방식)
- [5. Python을 이용한 테스트](#5-python을-이용한-테스트)
  - [5.1 환경 구성](#51-환경-구성)
  - [5.2 APIM 게이트웨이 정보 확인](#52-apim-게이트웨이-정보-확인)
  - [5.3 테스트 코드](#53-테스트-코드)
- [6. 참고 자료](#6-참고-자료)

---

## 1. API Management 인스턴스 생성

Azure Portal에서 API Management 인스턴스를 생성합니다. 가격 계층은 **Basic v2**를 사용합니다 (개발/테스트용, 몇 분 내 배포 완료).

### 절차

1. [Azure Portal](https://portal.azure.com/)에 로그인합니다.

2. 상단 검색창에 **API Management services**를 입력하고 선택합니다.

3. **+ 만들기(Create)** 를 클릭합니다.

4. **기본(Basics)** 탭에서 다음 정보를 입력합니다:

   | 항목 | 설명 | 예시 값 |
   |------|------|---------|
   | **구독(Subscription)** | 사용할 Azure 구독 | `My Subscription` |
   | **리소스 그룹(Resource group)** | 기존 그룹 선택 또는 새로 만들기 | `rg-foundry-krc` |
   | **지역(Region)** | 가까운 지역 선택 | `Korea Central` |
   | **리소스 이름(Resource name)** | APIM 인스턴스의 고유 이름 (도메인에 사용됨) | `apim-jyseong-demo` |
   | **조직 이름(Organization name)** | 조직 이름 | `My Organization` |
   | **관리자 이메일(Administrator email)** | 시스템 알림을 받을 이메일 | `admin@example.com` |
   | **가격 계층(Pricing tier)** | **Basic v2** 선택 | `Basic v2` |

5. **Managed Identity** 탭으로 이동합니다:
   - **시스템 할당 관리 ID(System assigned managed identity)** 를 **켜기(On)** 로 설정합니다.
   - > ⚠️ 이 설정은 [4.2 Managed Identity 방식](#42-방법-b--managed-identity-방식)에서 필요합니다. API Key 방식만 사용하더라도 향후 확장을 위해 활성화를 권장합니다.

6. **검토 + 만들기(Review + create)** → 유효성 검사 통과 후 **만들기(Create)** 를 클릭합니다.

7. 배포가 완료되면 **리소스로 이동(Go to resource)** 을 클릭합니다.

> 💡 **Tip**: Basic v2 계층은 일반적으로 **2~5분** 내에 배포가 완료됩니다. Developer 또는 Standard 계층은 30분 이상 소요될 수 있습니다.

---

## 2. Azure OpenAI 리소스 및 모델 배포

### 2.1 Azure OpenAI 리소스 생성

1. Azure Portal 상단 검색창에 **Azure OpenAI**를 입력하고 **Azure OpenAI**를 선택합니다.

2. **+ 만들기(Create)** 를 클릭합니다.

3. **기본(Basics)** 탭에서 다음 정보를 입력합니다:

   | 항목 | 설명 | 예시 값 |
   |------|------|---------|
   | **구독(Subscription)** | 사용할 Azure 구독 | `My Subscription` |
   | **리소스 그룹(Resource group)** | APIM과 동일한 리소스 그룹 권장 | `rg-foundry-krc` |
   | **지역(Region)** | 모델 가용 지역 선택 | `East US` |
   | **이름(Name)** | Azure OpenAI 리소스의 고유 이름 | `openai-jyseong-demo` |
   | **가격 계층(Pricing tier)** | Standard S0 | `Standard S0` |

   > ⚠️ **지역 선택 시 주의**: `gpt-4.1` 모델은 모든 지역에서 사용 가능하지 않습니다. [Azure OpenAI 모델 가용성 문서](https://learn.microsoft.com/en-us/azure/ai-services/openai/concepts/models)를 참고하여 해당 모델을 지원하는 지역을 선택하세요.

4. **네트워크(Networking)** 탭: 기본값(모든 네트워크 허용)을 유지합니다 (테스트 목적).

5. **검토 + 제출(Review + submit)** → **만들기(Create)** 를 클릭합니다.

6. 배포 완료 후 **리소스로 이동(Go to resource)** 을 클릭합니다.

7. 다음 정보를 메모합니다 (Azure Portal 또는 Foundry 포털 모두에서 확인 가능):
   - **엔드포인트(Endpoint)**: 개요 페이지 또는 **키 및 엔드포인트(Keys and Endpoint)** 메뉴에서 확인
     - 예: `https://openai-jyseong-demo.openai.azure.com/`
   - **API 키(Key)**: **키 및 엔드포인트** 메뉴에서 **키 1** 또는 **키 2**를 복사

### 2.2 모델 배포 (gpt-4.1)

1. Azure OpenAI 리소스 페이지에서 **모델 배포(Model deployments)** 메뉴를 클릭합니다.

2. **배포 관리(Manage Deployments)** 를 클릭하면 Azure AI Foundry 포털이 열립니다.

3. **+ 새 배포 만들기(Create new deployment)** 를 클릭합니다.

4. 다음 정보를 입력합니다:

   | 항목 | 설명 | 예시 값 |
   |------|------|---------|
   | **모델(Model)** | 배포할 모델 선택 | `gpt-4.1` |
   | **모델 버전(Model version)** | 기본값 또는 최신 안정 버전 | `Auto-update to default` |
   | **배포 이름(Deployment name)** | API 호출에 사용되는 ID (기억해두세요) | `gpt-4.1` |
   | **배포 유형(Deployment type)** | 표준(Standard) | `Standard` |

5. **만들기(Create)** 를 클릭합니다.

6. 배포가 완료되면, 배포 목록에서 해당 배포를 클릭하여 **세부 정보** 페이지로 이동합니다. 이 페이지에서 다음 정보를 확인할 수 있습니다:

   - **엔드포인트 → 대상 URI**: API 호출 시 사용할 엔드포인트 URL
     - 예: `https://openai-jyseong-demo.openai.azure.com/openai/deployments/gpt-4.1/...`
   - **엔드포인트 → 키**: Azure OpenAI API 키 (APIM Named Value에 저장할 값)
   - **배포 정보 → 이름**: 배포 이름 (예: `gpt-4.1`) — Python 코드의 `DEPLOYMENT` 변수에 사용
   - **배포 정보 → 모델 버전**: 예: `2025-04-14`

   > 💡 오른쪽 패널에서 **언어: Python**, **SDK: Azure OpenAI SDK**, **인증 유형: Key Auth**를 선택하면 샘플 코드를 직접 확인할 수 있습니다.

> 📋 **기록해야 할 정보 요약**
> 
> | 항목 | 예시 값 | 확인 위치 | 용도 |
> |------|---------|-----------|------|
> | OpenAI 엔드포인트 | `https://openai-jyseong-demo.openai.azure.com/` | 세부 정보 → 대상 URI 또는 Azure Portal → 키 및 엔드포인트 | APIM 백엔드 URL, Python 코드 |
> | API 키 | `••••••••••••••••` | 세부 정보 → 키 또는 Azure Portal → 키 및 엔드포인트 → 키 1/키 2 | APIM Named Value, API Key 인증 |
> | 배포 이름(Deployment ID) | `gpt-4.1` | 세부 정보 → 배포 정보 → 이름 | API 호출 시 모델 지정 |
> | API 버전 | `2024-10-21` | 사양 파일(`inference.json`)의 `info.version` 확인 | API 호출 시 버전 지정 |

---

## 3. API Management에 OpenAI API 사양 추가

APIM에 Azure OpenAI API를 등록하는 방법은 두 가지입니다. **방법 A** 또는 **방법 B** 중 하나를 선택하세요.

| 비교 항목 | 방법 A — OpenAPI 사양 파일 | 방법 B — Language Model API |
|-----------|--------------------------|---------------------------|
| **설정 난이도** | 중간 (사양 파일 다운로드·수정 필요) | 간단 (포털 위자드로 완료) |
| **세부 제어** | OpenAPI 사양의 모든 Operation 등록 | Chat Completions 엔드포인트 자동 구성 |
| **API 버전 선택** | 사양 파일 버전에 따라 자유롭게 선택 | APIM이 자동 구성 |
| **추가 정책 구성** | 인증 정책 수동 설정 필요 ([4장](#4-인증-구성) 참고) | 인증 키를 등록 과정에서 함께 설정, 토큰 관리·시맨틱 캐싱 정책도 위자드에서 구성 가능 |
| **권장 상황** | 모든 OpenAI Operation을 세밀하게 관리하려는 경우 | 빠르게 Chat Completions 위주로 테스트하려는 경우 |

---

### 3.1 방법 A — OpenAPI 사양 파일로 등록

#### 3.1.1 OpenAPI 사양 파일 다운로드 및 수정

1. 아래 링크에서 Azure OpenAI REST API의 OpenAPI 사양 파일(JSON)을 다운로드합니다:

   - **2024-10-21 GA 버전**: [inference.json (GitHub)](https://github.com/Azure/azure-rest-api-specs/blob/main/specification/cognitiveservices/data-plane/AzureOpenAI/inference/stable/2024-10-21/inference.json)
   
   > GitHub 페이지에서 **Raw** 버튼을 클릭하고 **다른 이름으로 저장**하여 `inference.json`으로 저장합니다.

2. 텍스트 에디터(VS Code 등)로 `inference.json` 파일을 엽니다.

3. 파일에서 `"servers"` 항목을 찾아 아래와 같이 수정합니다:

   **수정 전**:
   ```json
   "servers": [
     {
       "url": "https://{endpoint}/openai",
       "variables": {
         "endpoint": {
           "default": "your-resource-name.openai.azure.com"
         }
       }
     }
   ]
   ```

   **수정 후** (실제 Azure OpenAI 엔드포인트로 변경):
   ```json
   "servers": [
     {
       "url": "https://openai-jyseong-demo.openai.azure.com/openai",
       "variables": {
         "endpoint": {
           "default": "openai-jyseong-demo.openai.azure.com"
         }
       }
     }
   ]
   ```

   > ⚠️ `openai-jyseong-demo`를 **2.1**에서 생성한 Azure OpenAI 리소스 이름으로 교체하세요.

4. 파일을 저장합니다.

5. 파일 상단의 `info.version` 값을 확인합니다. 이 값이 API 호출 시 `api-version` 쿼리 파라미터로 사용됩니다.

   ```json
   "info": {
     "title": "Azure OpenAI Service API",
     "version": "2024-10-21"   ← 이 값을 기록
   }
   ```

   > ⚠️ **API 버전 선택 시 주의**
   >
   > | 구분 | 설명 | 예시 |
   > |------|------|------|
   > | **사양 파일 버전** (`info.version`) | 다운로드한 `inference.json` 파일에 명시된 버전 | `2024-10-21` |
   > | **Foundry 포털 샘플 코드 버전** | Foundry 포털 배포 세부 정보 페이지의 샘플 코드에 표시되는 버전 | `2024-12-01-preview` |
   >
   > **Python 코드의 `api_version` 파라미터는 사양 파일의 `info.version` 값과 일치시켜야 합니다.** APIM에 사양 파일을 등록하면, 해당 사양에 정의된 API 버전의 요청만 처리할 수 있기 때문입니다. 다른 버전을 사용하려면 해당 버전의 사양 파일을 다운로드하여 APIM에 등록하세요.

#### 3.1.2 APIM에 API 등록

1. Azure Portal에서 **API Management 인스턴스**로 이동합니다.

2. 왼쪽 메뉴에서 **APIs** → **+ Add API**를 클릭합니다.

3. **Define a new API** 섹션에서 **OpenAPI**를 선택합니다.

4. 다음 정보를 입력합니다:

   | 항목 | 설명 | 예시 값 |
   |------|------|---------|
   | **OpenAPI specification** | 3.1에서 수정한 `inference.json` 파일 업로드 또는 URL 입력 | 파일 업로드 |
   | **Display name** | API 표시 이름 | `Azure OpenAI API` |
   | **Name** | API 식별자 (자동 생성됨) | `azure-openai-api` |
   | **API URL suffix** | APIM 게이트웨이에서의 경로 접미사 (**반드시 `/openai`로 끝나야 함**) | `openai-api/openai` |

5. **Create**를 클릭합니다.

6. 생성 완료 후 왼쪽 API 목록에 **Azure OpenAI API**가 표시되며, Chat Completions, Completions 등 OpenAI의 각 작업(Operation)이 자동으로 가져와집니다.

> 💡 **Tip**: API URL suffix에 `/openai`를 포함시켜야 Python SDK의 `AzureOpenAI` 클라이언트가 올바른 경로로 요청을 보낼 수 있습니다.

---

### 3.2 방법 B — Language Model API로 등록

APIM의 **Language Model API** 기능을 사용하면 사양 파일 없이 포털 위자드만으로 API를 등록할 수 있습니다. 백엔드 리소스, 인증 헤더, 토큰 관리 정책까지 자동으로 구성됩니다.

#### 절차

1. Azure Portal에서 **API Management 인스턴스**로 이동합니다.

2. 왼쪽 메뉴에서 **APIs** → **+ Add API**를 클릭합니다.

3. **Create an AI API** 섹션에서 **Language Model API**를 선택합니다.

4. **Configure API** 탭에서 다음 정보를 입력합니다:

   | 항목 | 설명 | 예시 값 |
   |------|------|---------|
   | **Display name** | API 표시 이름 | `Azure OpenAI API` |
   | **Description** | API 설명 (선택 사항) | `Azure OpenAI gpt-4.1 via APIM` |
   | **LLM API URL** | Azure OpenAI의 엔드포인트 URL | `https://openai-jyseong-demo.openai.azure.com/openai` |
   | **Path** | APIM 게이트웨이에서의 경로 | `openai-api/openai` |
   | **API type** | **Create OpenAI API** 선택 | `Create OpenAI API` |
   | **Authorization header name** | 인증 헤더 이름 | `api-key` |
   | **API key** | Azure OpenAI API 키 ([2.2](#22-모델-배포-gpt-41)에서 확인한 키) | `xxxxxxxxxxxxxxxx` |

5. **Next**를 클릭합니다.

6. **Manage token consumption** 탭: 토큰 사용량 제한 정책을 설정합니다 (기본값 유지 가능).

7. **Apply semantic caching** 탭: 시맨틱 캐싱 정책을 설정합니다 (기본값 유지 가능).

8. **AI content safety** 탭: AI 콘텐츠 안전 정책을 설정합니다 (기본값 유지 가능).

9. **Review** → **Create**를 클릭합니다.

> 💡 **Tip**: Language Model API로 등록하면 다음이 **자동으로 구성**됩니다:
> - **Backend 리소스** 및 `set-backend-service` 정책
> - **Named Value**에 API 키 저장 및 인증 헤더 자동 전달
> - 토큰 관리, 시맨틱 캐싱, 콘텐츠 안전 등 AI 게이트웨이 정책
>
> 따라서 방법 B를 사용한 경우 **[4장 인증 구성](#4-인증-구성)의 API Key 방식 설정은 건너뛸 수 있습니다** (이미 위자드에서 설정 완료). Managed Identity 방식을 사용하려면 4.2를 참고하세요.

---

## 4. 인증 구성

APIM에서 Azure OpenAI 백엔드로의 인증 방식을 구성합니다. **API Key 방식**과 **Managed Identity 방식** 중 하나를 선택하여 적용하세요.

| 비교 항목 | API Key 방식 | Managed Identity 방식 |
|-----------|-------------|----------------------|
| **설정 난이도** | 간단 | 중간 |
| **보안 수준** | 보통 (키 노출 위험) | 높음 (키 없이 인증) |
| **키 관리** | 주기적 교체 필요 | 자동 관리 |
| **권장 환경** | 빠른 프로토타이핑, PoC | 프로덕션 환경 |

### 4.1 방법 A — API Key 방식

Azure OpenAI의 API 키를 APIM의 Named Value에 저장하고, Inbound 정책으로 요청 시 자동 전달합니다.

#### Step 1: Named Value에 API 키 저장

1. APIM 인스턴스 왼쪽 메뉴에서 **Named values**를 선택합니다.

2. **+ Add**를 클릭합니다.

3. 다음 정보를 입력합니다:

   | 항목 | 값 |
   |------|-----|
   | **Name** | `openai-api-key` |
   | **Display name** | `openai-api-key` |
   | **Type** | **Secret** |
   | **Value** | [2.2](#22-모델-배포-gpt-41)에서 확인한 Azure OpenAI API 키 (Foundry 포털 배포 세부 정보 → **키** 또는 Azure Portal → **키 및 엔드포인트** → **키 1/키 2**) |

4. **Save**를 클릭합니다.

> 💡 **보안 강화**: Key Vault에 API 키를 저장하고 Named Value에서 **Key Vault** 참조를 사용할 수도 있습니다. 프로덕션 환경에서는 Key Vault 참조를 권장합니다.

#### Step 2: Backend 리소스 생성 (선택 사항)

Backend 리소스를 생성하면 URL과 인증 정보를 한 곳에서 관리할 수 있습니다.

1. APIM 인스턴스 왼쪽 메뉴에서 **Backends**를 선택합니다.

2. **+ Add**를 클릭합니다.

3. 다음 정보를 입력합니다:

   | 항목 | 값 |
   |------|-----|
   | **Name** | `openai-backend` |
   | **Type** | **Custom URL** |
   | **Runtime URL** | `https://openai-jyseong-demo.openai.azure.com/openai` |
   | **Authorization credentials - Header** | 헤더 이름: `api-key`, 값: `{{openai-api-key}}` |

4. **Create**를 클릭합니다.

#### Step 3: Inbound 정책 구성

API에 정책을 추가하여 백엔드로 요청을 전달합니다. 두 가지 방법 중 하나를 선택합니다.

**방법 3-1: `set-backend-service` 정책 사용 (Backend 리소스 생성한 경우)**

1. APIM → **APIs** → **Azure OpenAI API** → **All operations** 선택

2. **Inbound processing** 섹션에서 **</>** (정책 편집기) 아이콘을 클릭합니다.

3. `<inbound>` 블록 안에 다음 정책을 추가합니다:

```xml
<inbound>
    <base />
    <set-backend-service backend-id="openai-backend" />
</inbound>
```

4. **Save**를 클릭합니다.

**방법 3-2: `set-header` 정책 사용 (Backend 리소스 없이 간단 구성)**

1. APIM → **APIs** → **Azure OpenAI API** → **All operations** 선택

2. **Inbound processing** 섹션에서 **</>** (정책 편집기) 아이콘을 클릭합니다.

3. `<inbound>` 블록 안에 다음 정책을 추가합니다:

```xml
<inbound>
    <base />
    <set-header name="api-key" exists-action="override">
        <value>{{openai-api-key}}</value>
    </set-header>
</inbound>
```

4. **Save**를 클릭합니다.

---

### 4.2 방법 B — Managed Identity 방식

APIM의 시스템 할당 관리 ID를 사용하여 API 키 없이 Azure OpenAI에 인증합니다. 프로덕션 환경에서 권장되는 방식입니다.

#### Step 1: Managed Identity 활성화 확인

1. APIM 인스턴스 왼쪽 메뉴에서 **Security** → **Managed identities**를 선택합니다.

2. **시스템 할당(System assigned)** 탭에서 상태가 **켜기(On)** 인지 확인합니다.
   - 꺼져 있다면 **켜기(On)** 로 변경 → **저장(Save)** 을 클릭합니다.

3. **개체 ID(Object ID)** 를 기록합니다 (역할 할당에 사용됩니다).

#### Step 2: Azure OpenAI 리소스에 역할 할당

1. Azure Portal에서 **Azure OpenAI 리소스**로 이동합니다.

2. 왼쪽 메뉴에서 **액세스 제어(IAM)** 를 선택합니다.

3. **+ 추가(Add)** → **역할 할당 추가(Add role assignment)** 를 클릭합니다.

4. **역할(Role)** 탭:
   - 검색창에 `Cognitive Services OpenAI User`를 입력합니다.
   - **Cognitive Services OpenAI User** 역할을 선택 → **다음(Next)** 을 클릭합니다.

5. **구성원(Members)** 탭:
   - **액세스 할당 대상**: **관리 ID(Managed identity)** 를 선택합니다.
   - **+ 구성원 선택(Select members)** 을 클릭합니다.
   - **관리 ID** 드롭다운에서 **API Management**를 선택합니다.
   - 목록에서 **1단계에서 생성한 APIM 인스턴스**(예: `apim-jyseong-demo`)를 선택합니다.
   - **선택(Select)** 을 클릭합니다.

6. **검토 + 할당(Review + assign)** 을 클릭합니다.

> ⚠️ 역할 할당이 적용되기까지 **최대 10분**이 소요될 수 있습니다.

#### Step 3: Inbound 정책 구성

1. APIM → **APIs** → **Azure OpenAI API** → **All operations** 선택

2. **Inbound processing** 섹션에서 **</>** (정책 편집기) 아이콘을 클릭합니다.

3. `<inbound>` 블록 안에 다음 정책을 추가합니다:

```xml
<inbound>
    <base />
    <authentication-managed-identity resource="https://cognitiveservices.azure.com" output-token-variable-name="managed-id-access-token" ignore-error="false" />
    <set-header name="Authorization" exists-action="override">
        <value>@("Bearer " + (string)context.Variables["managed-id-access-token"])</value>
    </set-header>
</inbound>
```

4. **Save**를 클릭합니다.

> 💡 **정책 설명**
> - `authentication-managed-identity`: APIM의 관리 ID를 사용하여 `https://cognitiveservices.azure.com` 리소스에 대한 액세스 토큰을 얻습니다.
> - `set-header`: 획득한 토큰을 `Authorization: Bearer <token>` 헤더에 설정하여 Azure OpenAI에 전달합니다.

---

## 5. Python을 이용한 테스트

### 5.1 환경 구성

#### Python 설치 확인

```bash
python --version
```

Python 3.8 이상이 필요합니다. 설치되어 있지 않다면 [python.org](https://www.python.org/downloads/)에서 다운로드합니다.

#### 가상 환경 생성 (권장)

```bash
# 가상 환경 생성
python -m venv .venv

# 가상 환경 활성화 (Windows PowerShell)
.\.venv\Scripts\Activate.ps1

# 가상 환경 활성화 (macOS/Linux)
source .venv/bin/activate
```

#### OpenAI SDK 설치

```bash
pip install openai
```

> `openai` 패키지 버전 **1.0 이상**이 필요합니다. 기존에 설치되어 있다면 `pip install --upgrade openai`로 업그레이드합니다.

### 5.2 APIM 게이트웨이 정보 확인

Python 코드에서 필요한 APIM 정보를 확인합니다.

#### Gateway URL 확인

1. Azure Portal → **APIM 인스턴스** → **개요(Overview)** 페이지

2. **Gateway URL** 값을 확인합니다.
   - 예: `https://apim-jyseong-demo.azure-api.net`

3. API URL suffix를 결합하면 최종 엔드포인트가 됩니다:
   - **최종 엔드포인트**: `https://apim-jyseong-demo.azure-api.net/openai-api`
   - > Python SDK에서 `azure_endpoint`에는 이 값을 입력합니다. SDK가 자동으로 `/openai/deployments/...` 경로를 추가합니다.

#### Subscription Key 확인

1. APIM 인스턴스 → 왼쪽 메뉴 **APIs** → **Subscriptions**를 선택합니다.

2. **Built-in all-access subscription** 행의 오른쪽 **...** 메뉴 → **Show/hide keys**를 클릭합니다.

3. **Primary key** 값을 복사합니다.

### 5.3 테스트 코드

아래 코드를 `test_openai_via_apim.py` 파일로 저장하고 실행합니다.

```python
import os
from openai import AzureOpenAI

# ============================================================
# 환경 변수 또는 직접 입력
# ============================================================
# APIM Gateway URL + API URL suffix (끝에 /openai 제외)
# 예: https://apim-jyseong-demo.azure-api.net/openai-api
APIM_ENDPOINT = os.environ.get("APIM_ENDPOINT", "https://<apim-name>.azure-api.net/<api-url-suffix>")

# APIM Subscription Key (Subscriptions 메뉴에서 확인)
APIM_SUBSCRIPTION_KEY = os.environ.get("APIM_SUBSCRIPTION_KEY", "<your-apim-subscription-key>")

# Azure OpenAI 배포 이름 (예: gpt-4.1)
DEPLOYMENT = os.environ.get("DEPLOYMENT", "gpt-4.1")

# Azure OpenAI API 버전 (사양 파일의 info.version 값과 일치시켜야 함)
API_VERSION = os.environ.get("API_VERSION", "2024-10-21")

# ============================================================
# API Key 인증 방식의 경우:
#   APIM 정책에서 api-key 헤더를 백엔드에 자동 전달하므로,
#   클라이언트에서는 APIM Subscription Key만 전달하면 됩니다.
#   api_key 파라미터에는 임의의 값을 넣어도 무방하지만,
#   APIM이 Subscription Key를 api-key 대신 사용하도록
#   default_headers로 전달합니다.
#
# Managed Identity 인증 방식의 경우:
#   APIM 정책에서 Bearer 토큰을 자동 생성하여 백엔드에 전달하므로,
#   클라이언트에서는 APIM Subscription Key만 전달하면 됩니다.
# ============================================================

client = AzureOpenAI(
    api_version=API_VERSION,
    azure_endpoint=APIM_ENDPOINT,
    api_key="placeholder",  # APIM 정책이 백엔드 인증을 처리하므로 실제 OpenAI 키 불필요
    default_headers={
        "Ocp-Apim-Subscription-Key": APIM_SUBSCRIPTION_KEY
    },
)

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

#### 환경 변수로 실행하기 (권장)

코드에 키를 직접 입력하는 대신, 환경 변수를 설정하여 실행할 수 있습니다.

**Windows PowerShell**:
```powershell
$env:APIM_ENDPOINT = "https://apim-jyseong-demo.azure-api.net/openai-api"
$env:APIM_SUBSCRIPTION_KEY = "your-subscription-key-here"
$env:DEPLOYMENT = "gpt-4.1"
$env:API_VERSION = "2024-10-21"

python test_openai_via_apim.py
```

**macOS / Linux (Bash)**:
```bash
export APIM_ENDPOINT="https://apim-jyseong-demo.azure-api.net/openai-api"
export APIM_SUBSCRIPTION_KEY="your-subscription-key-here"
export DEPLOYMENT="gpt-4.1"
export API_VERSION="2024-10-21"

python test_openai_via_apim.py
```

#### 예상 출력

```
============================================================
응답 내용:
안녕하세요! 저는 AI 어시스턴트입니다. 질문에 답하거나 정보를 제공하는 등 다양한 방법으로 도움을 드릴 수 있습니다. 무엇을 도와드릴까요?
============================================================
모델: gpt-4.1
사용 토큰 - 프롬프트: 24, 완료: 52, 총: 76
```

---

## 6. 참고 자료

| 항목 | 링크 |
|------|------|
| API Management 인스턴스 생성 | https://learn.microsoft.com/en-us/azure/api-management/get-started-create-service-instance |
| Azure OpenAI API를 APIM에 가져오기 | https://learn.microsoft.com/en-us/azure/api-management/azure-openai-api-from-specification |
| OpenAPI 사양으로 API 가져오기 | https://learn.microsoft.com/en-us/azure/api-management/import-api-from-oas |
| Language Model API 가져오기 | https://learn.microsoft.com/en-us/azure/api-management/openai-compatible-llm-api |
| LLM API 인증/권한 부여 | https://learn.microsoft.com/en-us/azure/api-management/api-management-authenticate-authorize-ai-apis |
| Azure OpenAI 모델 가용성 | https://learn.microsoft.com/en-us/azure/ai-services/openai/concepts/models |
| OpenAI Python SDK | https://github.com/openai/openai-python |
| Azure OpenAI REST API 사양 (GitHub) | https://github.com/Azure/azure-rest-api-specs/tree/main/specification/cognitiveservices/data-plane/AzureOpenAI/inference/stable |
| APIM AI 게이트웨이 기능 | https://learn.microsoft.com/en-us/azure/api-management/genai-gateway-capabilities |
