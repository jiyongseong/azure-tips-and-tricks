# Azure Logic App에서 HTTP 액션으로 Translator API 호출하기 (Managed Identity를 이용한 인증)

Microsoft Azure Cognitive Services Translator 서비스를 Logic App에서 사용하는 방법은 Connector를 사용하는 것이다.

현재(2026년 2월 기준)는 v3(미리보기)까지 제공되고 있으며, v2 버전도 여전히 유효하다.

<img src='./images/microsoft-translator-connector.png'>

v3 버전은 v2에 비해서 더 많은 기능들을 제공한다(너무 당연하다!)

<img src='./images/microsoft-translator-v3-connector.png'>


> v3 connector에 대해서는 [Microsoft Translator V3(미리 보기)](https://learn.microsoft.com/ko-kr/connectors/microsofttranslatorv/)를 참고

설정이 매우 간편해서 Translator와 같이 잘 준비된 서비스들은 connector를 이용하는 편인데, 오류가 발생되는 경우 debugging이 용이하지 않다는 **치명적인** 단점이 존재한다.

근래(2026년 1~2월 기준)에 다음과 같이 알수 없는(찾지 못한) internal server error를 몇 차례 겪다보니, 대안을 찾아야만 했다.

<img src='./images/microsoft-translator-connector-error.png'>

가장 간편한 대안은 http로 해당 서비스를 직접 호출하는 것이다.

설정은 매우 간단하다.

http action을 추가하고, 다음과 같이 설정하면 된다.

<img src='./images/logicapp-headers.png'>


* URL : 
    * 기본적으로 다음과 같이 사용하였다.
        * https://api.cognitive.microsofttranslator.com/translate?api-version=3.0&to=ko&textType=html
    * translator API의 spec에 대해서는 다음의 링크를 참고하여 URL을 조합하면 된다.
        * https://learn.microsoft.com/en-us/azure/ai-services/translator/text-translation/reference/rest-api-guide
* Method : POST
* Headers에는 다음의 값들을 추가한다.
    * Ocp-Apim-Subscription-Region : trnaslator API가 생성된 지역의 이름을 기술한다.
    * Content-Type : application/json
    * Ocp-Apim-ResourceId : translator 리소스의 리소스 ID를 입력한다([Translator 리소스의 리소스 ID 찾기](./how-to-call-translator-api-via-http-in-logic-apps-managedid.md#translator-리소스의-리소스-id-찾기) 참고).
* Authentication 정보 입력.
    * Advanced parameters의 드롭다운 메뉴를 클릭하고, Authentication 체크 박스를 클릭.
    <img src='./images/translator-authentication.png'>
    * 하단에 인증 관련 입력 상자가 나타나면, 다음의 정보를 입력
    <img src='./images/translator-authentication-values.png'>
        * Authentication type : Managed identity
        * Managed identity : System-assigned managed identity
        * Audience : https://cognitiveservices.azure.com
    
* Body : 번역하려는 text를 지정. 

## Translator 리소스의 리소스 ID 찾기
Azure 포털에서 translator 리소스로 이동하고, 좌측 메뉴에서 Resource Management > Properties로 이동.

해당 메뉴에서 우측 pane의 Resource ID의 값을 복사.

<img src='./images/translator-resource-id.png'>

✍️ 2026년 2월 19일 씀.