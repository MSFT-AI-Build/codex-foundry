# 모듈 2: Foundry 모델 준비

[⬅️ 이전: 아키텍처와 사전 준비](../01-architecture-and-prerequisites/) | [🏠 메인](../README.md) | [➡️ 다음: APIM AI Gateway](../03-apim-gateway/)

## 학습 목표

- Responses API를 지원하는 모델을 배포합니다.
- endpoint와 배포명을 구분합니다.
- APIM 연결 전에 Foundry backend 상태를 확인합니다.

예상 시간은 15~25분입니다.

## 2.1 모델과 배포명 선택

Microsoft Foundry portal에서 프로젝트 또는 Azure OpenAI 리소스를 열고 Responses API를 지원하는 모델을 배포합니다.

확인할 항목은 다음과 같습니다.

1. 배포 지역에서 Responses API가 지원되는가?
2. 선택한 모델과 버전이 해당 지역에서 제공되는가?
3. 배포 quota가 실습 요청을 처리할 만큼 있는가?
4. 배포명이 Codex 설정의 `model` 값과 정확히 일치하는가?

예를 들어 모델 이름이 `gpt-5-codex`여도 배포명을 `gpt-codex-prod`로 만들었다면 Codex에는 다음과 같이 설정합니다.

```toml
model = "gpt-codex-prod"
```

## 2.2 endpoint 기록

새 v1 계약의 기본 endpoint는 다음 형태입니다.

```text
https://RESOURCE_NAME.openai.azure.com/openai/v1/
```

요청 URL은 다음과 같습니다.

```text
POST https://RESOURCE_NAME.openai.azure.com/openai/v1/responses
```

이 값은 APIM backend에서만 사용합니다. 개발자의 Codex 설정에는 Foundry endpoint나 Foundry API key를 배포하지 않습니다.

## 2.3 Managed Identity 준비

Azure Portal에서 APIM 리소스를 열어 `Identity`의 system-assigned identity를 활성화합니다. 이어서 Foundry/Azure OpenAI 리소스의 Access control(IAM)에서 이 identity에 모델 inference 권한을 부여합니다.

역할 할당은 전파에 몇 분이 걸릴 수 있습니다. 바로 `401` 또는 `403`이 발생하면 policy를 반복 수정하기 전에 역할 전파 시간을 고려합니다.

### 토큰 대상(resource/audience)

현재 Foundry OpenAI v1 문서는 Entra ID scope로 `https://ai.azure.com/.default`를 안내합니다. APIM의 `authentication-managed-identity` policy에서는 scope의 `/.default`를 제외한 다음 resource 값을 사용합니다.

```text
https://ai.azure.com
```

기존 Azure OpenAI endpoint가 `https://cognitiveservices.azure.com` audience를 요구하는 환경도 있습니다. 이 경우 실제 리소스 문서와 성공한 token audience를 기준으로 APIM policy를 조정하고, 두 값을 임의로 혼용하지 않습니다.

## 2.4 선택 사항: backend 직접 smoke test

조직 정책상 허용되고 임시 credential을 사용할 수 있을 때만 APIM 구성 전에 직접 테스트합니다. 아래 예시는 현재 PowerShell 프로세스의 환경 변수만 사용합니다.

```powershell
$headers = @{
    'api-key'      = $env:AZURE_OPENAI_API_KEY
    'Content-Type' = 'application/json'
}

$body = @{
    model = 'YOUR_DEPLOYMENT_NAME'
    input = 'Reply with FOUNDRY_BACKEND_OK.'
} | ConvertTo-Json

Invoke-RestMethod `
    -Method Post `
    -Uri 'https://YOUR_RESOURCE_NAME.openai.azure.com/openai/v1/responses' `
    -Headers $headers `
    -Body $body
```

이 단계의 API key는 로컬 진단용입니다. 워크숍의 최종 구조에서는 APIM Managed Identity가 backend에 인증합니다.

## 체크포인트

- [ ] Responses API 지원 모델을 배포했다.
- [ ] 모델 이름과 배포명을 구분해 기록했다.
- [ ] Foundry v1 endpoint를 기록했다.
- [ ] APIM Managed Identity를 활성화했다.
- [ ] Managed Identity에 최소 inference 역할을 부여했다.

## 참고 자료

- [Microsoft Learn: Use the Azure OpenAI Responses API](https://learn.microsoft.com/azure/foundry/openai/how-to/responses)
- [Microsoft Learn: Role-based access control for Microsoft Foundry](https://learn.microsoft.com/azure/foundry/concepts/rbac-foundry)

[⬅️ 이전: 아키텍처와 사전 준비](../01-architecture-and-prerequisites/) | [🏠 메인](../README.md) | [➡️ 다음: APIM AI Gateway](../03-apim-gateway/)
