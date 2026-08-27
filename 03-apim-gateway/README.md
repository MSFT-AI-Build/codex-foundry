# 모듈 3: APIM AI Gateway 구성

[⬅️ 이전: Foundry 모델 준비](../02-foundry-model/) | [🏠 메인](../README.md) | [➡️ 다음: Codex 설정](../04-codex-configuration/)

## 학습 목표

- Codex 전용 Responses API operation을 만듭니다.
- APIM subscription과 Managed Identity를 서로 다른 인증 계층으로 구성합니다.
- SSE streaming을 보존하는 policy를 적용합니다.

예상 시간은 25~35분입니다.

## 3.1 공개 API 계약 생성

APIM에서 API를 하나 생성하거나 기존 AI API에 operation을 추가합니다.

| 설정 | 값 |
| --- | --- |
| Display name | `Codex Foundry Gateway` |
| URL scheme | `HTTPS` |
| API URL suffix | `codex/openai/v1` |
| Operation method | `POST` |
| Operation URL | `/responses` |
| Subscription required | `Yes` |

완성된 공개 URL은 다음과 같습니다.

```text
https://APIM_HOST/codex/openai/v1/responses
```

APIM product를 만들고 이 API를 연결한 뒤, 실습 사용자에게 product subscription을 발급합니다. 이 key는 APIM client 인증용이며 Foundry API key와 다른 값입니다.

## 3.2 backend 인증 policy 적용

[APIM policy 템플릿](../templates/apim-policy.xml)을 API 또는 operation scope에 붙여 넣고 `YOUR_RESOURCE_NAME`을 실제 값으로 바꿉니다.

핵심 policy는 다음과 같습니다.

```xml
<set-backend-service base-url="https://YOUR_RESOURCE_NAME.openai.azure.com" />
<rewrite-uri template="/openai/v1/responses" copy-unmatched-params="false" />
<authentication-managed-identity
  resource="https://ai.azure.com"
  output-token-variable-name="foundry-token"
  ignore-error="false" />
<set-header name="Authorization" exists-action="override">
  <value>@("Bearer " + (string)context.Variables["foundry-token"])</value>
</set-header>
```

정책의 책임은 네 가지입니다.

1. 공개 path를 Foundry v1 Responses path로 rewrite합니다.
2. APIM Managed Identity token을 획득합니다.
3. backend Authorization header를 설정합니다.
4. Codex의 SSE 응답을 버퍼링하지 않고 전달합니다.

> 기존 Azure OpenAI 리소스가 Cognitive Services audience를 요구하면 `resource`를 `https://cognitiveservices.azure.com`으로 바꿉니다. `401` 진단 시 role assignment와 함께 token audience를 확인하세요.

## 3.3 backend 및 secret 경계 확인

APIM trace에서 다음을 확인합니다.

| 구간 | 기대값 |
| --- | --- |
| Client → APIM | `Ocp-Apim-Subscription-Key` 존재 |
| APIM inbound | subscription 검증 성공 |
| APIM → Foundry | Bearer token 존재 |
| Backend URL | `/openai/v1/responses` |
| Request body | `model`이 실제 배포명 |
| Response | `text/event-stream` 또는 정상 JSON |

subscription key가 backend로 전달되지 않도록 policy에서 `Ocp-Apim-Subscription-Key`를 삭제합니다. 반대로 Foundry credential이나 Managed Identity token은 client 응답과 로그에 노출하지 않습니다.

## 3.4 APIM REST smoke test

먼저 현재 PowerShell 세션에 key를 넣습니다.

```powershell
$env:CODEX_APIM_SUBSCRIPTION_KEY = 'YOUR_APIM_SUBSCRIPTION_KEY'
```

그 다음 APIM endpoint를 직접 호출합니다.

```powershell
$headers = @{
    'Ocp-Apim-Subscription-Key' = $env:CODEX_APIM_SUBSCRIPTION_KEY
    'Content-Type'              = 'application/json'
}

$body = @{
    model  = 'YOUR_DEPLOYMENT_NAME'
    input  = 'Reply with APIM_OK.'
    stream = $false
} | ConvertTo-Json

Invoke-RestMethod `
    -Method Post `
    -Uri 'https://YOUR_APIM_HOST/codex/openai/v1/responses' `
    -Headers $headers `
    -Body $body
```

이 테스트가 실패하면 Codex 설정으로 넘어가지 않습니다. APIM trace에서 client 인증, rewrite, Managed Identity, Foundry 응답 순서로 원인을 분리합니다.

이어서 실제 SSE streaming이 중간에 버퍼링되지 않는지 확인합니다. `curl.exe`의 `-N` 옵션은 수신 데이터를 버퍼링하지 않고 도착하는 즉시 출력합니다.

```powershell
$streamBody = @{
    model  = 'YOUR_DEPLOYMENT_NAME'
    input  = 'Count from one to five, one number at a time.'
    stream = $true
} | ConvertTo-Json -Compress

$streamBody | curl.exe -N `
    -X POST `
    'https://YOUR_APIM_HOST/codex/openai/v1/responses' `
    -H "Ocp-Apim-Subscription-Key: $env:CODEX_APIM_SUBSCRIPTION_KEY" `
    -H 'Content-Type: application/json' `
    -H 'Accept: text/event-stream' `
    --data-binary '@-'
```

여러 SSE event가 응답 완료 후 한꺼번에 출력되는 것이 아니라 생성되는 동안 순차적으로 보여야 하며, 마지막에 `response.completed` event가 도착해야 합니다. 출력이 끝날 때까지 아무 내용도 보이지 않는다면 APIM policy, gateway 또는 중간 네트워크 장비의 buffering을 확인합니다.

## 3.5 운영 정책 선택 사항

연결 성공 후에만 다음 정책을 단계적으로 추가합니다.

- subscription 또는 사용자별 token rate limit
- request ID/correlation ID 기록
- token metric을 Azure Monitor로 전송
- backend pool과 circuit breaker
- prompt/response content를 제외한 메타데이터 로깅

콘텐츠 로깅은 코드와 prompt를 수집할 수 있으므로 기본값을 꺼 둡니다. 필요한 경우 보안·개인정보·보존 기간 승인을 먼저 받습니다.

## 체크포인트

- [ ] `POST /codex/openai/v1/responses` operation이 생성되었다.
- [ ] API가 subscription을 요구한다.
- [ ] Managed Identity가 backend 인증에 사용된다.
- [ ] public path가 Foundry v1 path로 rewrite된다.
- [ ] APIM 직접 호출이 성공한다.
- [ ] SSE event가 생성되는 동안 순차적으로 도착하고 `response.completed`로 끝난다.
- [ ] SSE를 버퍼링하는 변환/캐시 policy가 없다.

## 참고 자료

- [Microsoft Learn: AI gateway capabilities in APIM](https://learn.microsoft.com/azure/api-management/genai-gateway-capabilities)
- [Microsoft Learn: Authenticate with managed identity policy](https://learn.microsoft.com/azure/api-management/authentication-managed-identity-policy)

[⬅️ 이전: Foundry 모델 준비](../02-foundry-model/) | [🏠 메인](../README.md) | [➡️ 다음: Codex 설정](../04-codex-configuration/)
