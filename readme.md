# Azure Tips & Tricks 🧰⚡

실전에서 만난 Azure 오류/제약/우회 설정/숨은 팁을 Azure Resource(서비스)별로 정리하고 공유하는 저장소입니다.

이 저장소는 Azure를 사용하면서 겪는 다양한 오류(Error) / 예외 상황 / 운영 꼼수 / 공식 문서에 잘 나오지 않는 설정 방법 / 트러블슈팅 노하우를 Azure 리소스(서비스) 단위로 구조화해 기록하고 공유하기 위한 공간입니다.

✅ Tip의 성격상, 공식 문서(정식 가이드)와 다른 "현장 기반 우회/경험 기반 해결"이 포함될 수 있습니다.

✅ 가능한 경우 공식 문서 링크를 함께 제공해 "공식 권장"과 "현장 팁/workaround"를 구분할 예정입니다.


## 🎯 목표

* Azure 서비스별로 자주 발생하는 오류/이슈를 빠르게 검색하고 해결
* 운영/구축 과정에서 문서에 드러나지 않는 디테일(권한, 네트워크, 제한사항 등) 축적
* 고객/내부 지원 과정에서 재사용 가능한 재현 가능한 트러블슈팅 아카이브 구축


## 👥 대상

* Azure를 운영/구축/마이그레이션하는 엔지니어, 아키텍트, SRE/DevOps
* 고객 이슈 대응 중 빠르게 “원인-해결-검증”을 찾고 싶은 담당자
* 특정 리소스의 엣지 케이스(정책, 권한, 네트워크, SKU/리전 제한 등)를 정리하고 싶은 사용자


## 🗂️ 폴더 구조 (목표로 하는 구조. 변경될 예정입니다.) 

    azure-tips-and-tricks/
    ├─ APIM/                     # API Management 등
    ├─ identity/                 # Entra ID, RBAC, Managed Identity 등
    ├─ networking/               # VNet, Private Endpoint, DNS, FW, LB 등
    ├─ compute/                  # VM, VMSS, AKS, App Service 등
    ├─ storage/                  # Storage Account, Blob/File, ADLS Gen2 등
    ├─ database/                 # Azure SQL, MySQL, PostgreSQL 등
    ├─ monitoring/               # Azure Monitor, Log Analytics, AMA 등
    ├─ security/                 # Defender for Cloud, Key Vault 등
    ├─ governance/               # Policy, Cost/Tagging 등
    ├─ devops/                   # GitHub Actions, Azure DevOps, IaC 등
    └─ ai-ml/                    # Azure AI, OpenAI, AML 등

### APIM
* APIM을 통한 API Import 가이드
    * [Azure OpenAI API Import 가이드](./APIM/importing-aoai-and-msft-foundry/APIM을_통한_Azure_OpenAI_API_import_가이드.md)
    * [Microsoft Foundry API Import 가이드](./APIM/importing-aoai-and-msft-foundry/APIM을_통한_MSFT_Foundry_API_Import_가이드.md)

### Compute
* Azure Logic App에서 HTTP 액션으로 Translator API 호출하기
    * [Key를 이용한 인증](./compute/how-to-call-translator-api-via-http-in-logic-apps.md)
    * [Managed Identity를 이용한 인증](./compute/how-to-call-translator-api-via-http-in-logic-apps-managedid.md)

## 🔎 사용 방법

* 문제가 발생한 Azure 서비스(리소스) 폴더로 이동합니다.
* 파일 제목(또는 태그/키워드)로 증상/에러코드를 찾습니다.
* 각 문서는 다음 흐름을 따르는 것을 권장합니다.
    > Symptoms → Root cause → Resolution → Verification → References

## 🤝 기여 방법 (Contributing)
이슈/오타/개선/새로운/나만 알고 있는 Tip 추가 모두 환영합니다!

* Issue로 증상/요구 사항을 남깁니다.
* PR로 문서 추가/수정합니다.
* PR에는 가능하면 다음을 포함해 주세요.
    * 증상(에러 메시지/코드)
    * 원인 추정
    * 해결 절차
    * 검증 방법
    * (가능하면) 공식 문서 링크

Tip은 "정답"이 하나가 아닐 수 있습니다.

대신, **언제 유효하고 언제 위험한지**를 명확히 기술 해주시면 더 좋습니다.

## ⚠️ Disclaimer

* 이 저장소의 내용은 개인/현장 경험 기반의 팁이며, 모든 환경에서 동일하게 동작함을 보장하지 않습니다.

* 보안/컴플라이언스 요구사항이 있는 환경에서는 반드시 조직 정책을 우선합니다.

* **해당 저장소의 내용은 Microsoft의 공식 문서 또는 공식 가이드가 아닙니다.**

## 📜 License
이 프로젝트는 누구나 자유롭게 공유하고 활용할 수 있습니다.

## ✨ Maintainer
* [Ji Yong Seong(MSFT)](https://github.com/jiyongseong)