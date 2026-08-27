# 모듈 4: Codex 연결 설정

[⬅️ 이전: APIM AI Gateway](../03-apim-gateway/) | [🏠 메인](../README.md) | [➡️ 다음: Production 전환 체크리스트](../05-production-checklist/)

## 학습 목표

- APIM을 Codex custom model provider로 등록합니다.
- secret을 환경 변수로 주입합니다.
- 기본 provider 또는 선택 프로필로 실행합니다.

예상 시간은 15분입니다.

## 4.1 설정 위치 이해

Windows에서 사용자 전역 설정의 기본 위치는 다음과 같습니다.

```text
%USERPROFILE%\.codex\config.toml
```

`CODEX_HOME`을 별도로 지정했다면 `%CODEX_HOME%\config.toml`을 사용합니다.

`model_provider`와 `model_providers`는 machine-local provider/auth 설정이므로 저장소의 `.codex/config.toml`이 아니라 **사용자 전역 설정**에 둡니다. Codex는 신뢰된 프로젝트의 project config를 읽지만, project config에서 provider와 인증 관련 키를 재정의하지 않습니다.

## 4.2 APIM subscription key 저장

현재 PowerShell 세션에서만 사용하려면 다음을 실행합니다.

```powershell
$env:CODEX_APIM_SUBSCRIPTION_KEY = 'YOUR_APIM_SUBSCRIPTION_KEY'
```

Windows 사용자 환경 변수로 저장하려면 다음을 실행하고 새 PowerShell 창을 엽니다.

```powershell
[Environment]::SetEnvironmentVariable(
    'CODEX_APIM_SUBSCRIPTION_KEY',
    'YOUR_APIM_SUBSCRIPTION_KEY',
    'User'
)
```

값을 출력하지 않고 존재 여부만 확인합니다.

```powershell
if ([string]::IsNullOrWhiteSpace($env:CODEX_APIM_SUBSCRIPTION_KEY)) {
    'MISSING'
} else {
    'SET'
}
```

## 4.3 사용자 전역 provider 등록

기존 설정을 먼저 백업하고 편집합니다.

```powershell
New-Item -ItemType Directory -Force "$env:USERPROFILE\.codex" | Out-Null

$configPath = "$env:USERPROFILE\.codex\config.toml"
if (Test-Path $configPath) {
    Copy-Item $configPath "$configPath.backup"
}

notepad $configPath
```

기존 설정을 보존하면서 다음 내용을 병합합니다. `base_url` 끝에 `/responses`를 넣지 않습니다.

```toml
model_provider = "foundry_apim"
model = "YOUR_DEPLOYMENT_NAME"

[model_providers.foundry_apim]
name = "Microsoft Foundry via APIM"
base_url = "https://YOUR_APIM_HOST/codex/openai/v1"
wire_api = "responses"
requires_openai_auth = false
env_http_headers = { "Ocp-Apim-Subscription-Key" = "CODEX_APIM_SUBSCRIPTION_KEY" }
request_max_retries = 4
stream_max_retries = 5
stream_idle_timeout_ms = 300000
```

Codex가 만드는 최종 URL은 다음과 같습니다.

```text
https://YOUR_APIM_HOST/codex/openai/v1/responses
```

`env_http_headers`의 왼쪽은 HTTP header 이름이고 오른쪽은 환경 변수 이름입니다. 오른쪽에 실제 key 값을 쓰지 않습니다.

## 4.4 선택 사항: Foundry 전용 프로필

Foundry를 항상 기본 provider로 사용하지 않으려면 provider 정의는 전역 `config.toml`에 두고, 선택값을 별도 프로필 파일에 둡니다.

```toml
# %USERPROFILE%\.codex\foundry.config.toml
model_provider = "foundry_apim"
model = "YOUR_DEPLOYMENT_NAME"
```

프로필을 선택해 실행합니다.

```powershell
codex --profile foundry
codex exec --profile foundry '한 문장으로 연결 성공이라고 답해.'
```

현재 Codex 문서의 프로필 형식은 `$CODEX_HOME/profile-name.config.toml`입니다. 예전 `[profiles.foundry]` table 예시와 혼용하지 않습니다.

## 기존 Preview `api-version` 계약

기존 APIM이 다음 URL만 제공한다면 Preview 설정을 사용합니다.

```text
POST https://YOUR_APIM_HOST/route/openai/responses?api-version=YOUR_API_VERSION
```

```toml
model_provider = "foundry_apim_preview"
model = "YOUR_DEPLOYMENT_NAME"

[model_providers.foundry_apim_preview]
name = "Microsoft Foundry via APIM (preview contract)"
base_url = "https://YOUR_APIM_HOST/route/openai"
wire_api = "responses"
requires_openai_auth = false
env_http_headers = { "Ocp-Apim-Subscription-Key" = "CODEX_APIM_SUBSCRIPTION_KEY" }
query_params = { api-version = "YOUR_API_VERSION" }
request_max_retries = 4
stream_max_retries = 5
stream_idle_timeout_ms = 300000
```

Preview contract에서는 APIM rewrite도 해당 Azure API version의 backend path에 맞춰야 합니다. v1 path와 Preview query parameter를 함께 사용하지 않습니다.

## 4.5 다른 subscription header 이름

APIM API에서 `api-key` 같은 다른 header 이름을 사용한다면 다음 한 줄만 실제 계약에 맞게 바꿉니다.

```toml
env_http_headers = { "api-key" = "CODEX_APIM_SUBSCRIPTION_KEY" }
```

## 4.6 Codex 연결 검증

먼저 현재 Codex가 설정 파일과 provider 항목을 정상적으로 읽는지 확인합니다.

```powershell
codex doctor --summary
```

문제가 있으면 secret 값을 노출하지 않는 JSON 진단 결과를 확인합니다.

```powershell
codex doctor --json
```

기본 provider로 등록했다면 짧은 비대화형 요청을 실행합니다.

```powershell
codex exec '한 문장으로 연결 성공이라고 답해.'
```

Foundry 전용 프로필을 사용한다면 해당 프로필을 명시합니다.

```powershell
codex exec --profile foundry '한 문장으로 연결 성공이라고 답해.'
```

마지막으로 대화형 UI를 실행하고 `/status`에서 provider가 `foundry_apim`, model이 실제 Foundry 배포명으로 표시되는지 확인합니다. 프로필 구성을 사용한다면 `codex --profile foundry`로 실행합니다.

```powershell
codex
```

`doctor`는 로컬 설정과 실행 환경을 진단하고, 실제 end-to-end 연결 성공 여부는 `codex exec` 응답과 APIM trace를 함께 확인해 판단합니다.

## 체크포인트

- [ ] provider 설정이 사용자 전역 `config.toml`에 있다.
- [ ] `base_url`에 `/responses`가 중복되지 않는다.
- [ ] `model`이 Foundry 배포명과 일치한다.
- [ ] `wire_api = "responses"`를 사용한다.
- [ ] secret 값이 TOML이나 Git 파일에 없다.
- [ ] v1과 Preview 계약 중 하나만 선택했다.
- [ ] `codex doctor --summary`에 provider 설정 오류가 없다.
- [ ] `codex exec`가 APIM을 통해 정상 응답을 반환한다.
- [ ] `/status`에서 의도한 provider와 배포명이 선택되어 있다.

## 참고 자료

- [OpenAI Docs: Configuration reference](https://developers.openai.com/codex/config-reference)
- [OpenAI Docs: Advanced configuration](https://developers.openai.com/codex/config-advanced)
- [전체 설정 템플릿](../templates/config.toml.example)

[⬅️ 이전: APIM AI Gateway](../03-apim-gateway/) | [🏠 메인](../README.md) | [➡️ 다음: Production 전환 체크리스트](../05-production-checklist/)
