# 모듈 1: 아키텍처와 사전 준비

[🏠 메인](../README.md) | [➡️ 다음: Foundry 모델 준비](../02-foundry-model/)

## 학습 목표

- Codex, APIM, Foundry 사이의 책임 경계를 설명합니다.
- 실습에 필요한 값과 권한을 준비합니다.
- v1 계약과 기존 Preview 계약 중 하나를 선택합니다.

예상 시간은 10분입니다.

## 1.1 구성 요소의 역할

| 구성 요소 | 역할 | 보유하는 인증 정보 |
| --- | --- | --- |
| Codex CLI | Responses API client | APIM subscription key |
| APIM | 인증, 라우팅, 제한, 관측 | Managed Identity |
| Microsoft Foundry | 모델 추론 | 모델 배포와 quota |

권장 경계는 Codex가 Foundry credential을 직접 가지지 않는 것입니다. 개발자에게는 APIM 접근 권한만 배포하고, APIM이 Managed Identity로 Foundry에 인증합니다.

```text
Developer workstation                 Azure boundary
┌──────────────────┐                 ┌────────────────────────────┐
│ Codex            │ subscription   │ APIM                       │
│ config.toml      ├──── key ──────►│ auth / quota / logging     │
│ env var (secret) │                 └─────────────┬──────────────┘
└──────────────────┘                               │ Managed Identity
                                                   ▼
                                      ┌────────────────────────────┐
                                      │ Foundry model deployment   │
                                      │ OpenAI v1 Responses API    │
                                      └────────────────────────────┘
```

## 1.2 API 계약 선택

### 권장: OpenAI v1 계약

```text
APIM public base URL : https://APIM_HOST/codex/openai/v1
Codex final request  : POST .../codex/openai/v1/responses
Foundry backend      : POST https://RESOURCE.openai.azure.com/openai/v1/responses
```

v1 endpoint에는 `api-version` query parameter를 추가하지 않습니다.

### 기존 환경: Preview 계약

```text
APIM public base URL : https://APIM_HOST/route/openai
Codex final request  : POST .../route/openai/responses?api-version=API_VERSION
```

Preview 계약은 `api-version` query parameter를 사용하는 기존 APIM 환경을 위한 경로입니다. 새 실습에서는 v1 계약을 권장합니다.

## 1.3 입력값 시트

다음 표를 복사해 실제 값으로 채웁니다. secret은 값 자체가 아니라 저장 위치만 기록합니다.

| 이름 | 예시 | 내 환경 |
| --- | --- | --- |
| Azure subscription | `Contoso-Dev` | |
| Resource group | `rg-codex-foundry` | |
| Foundry resource name | `contoso-foundry` | |
| Foundry endpoint | `https://contoso-foundry.openai.azure.com` | |
| Model deployment name | `gpt-codex-prod` | |
| APIM service name | `contoso-ai-apim` | |
| APIM hostname | `contoso-ai-apim.azure-api.net` | |
| APIM public base path | `/codex/openai/v1` | |
| APIM product/subscription | `codex-developers` | |
| Secret 저장 위치 | 사용자 환경 변수 | |

> `model`에는 모델 카탈로그 이름이 아니라 Foundry에서 생성한 **배포명**을 사용합니다.

## 1.4 권한 확인

최소한 다음 작업을 수행할 수 있어야 합니다.

- Foundry 모델 배포 조회 또는 생성
- APIM API와 policy 편집
- APIM system-assigned Managed Identity 활성화
- Foundry 리소스 범위에 모델 호출 역할 할당
- APIM product/subscription 생성 또는 기존 key 발급

조직 정책 때문에 역할 할당을 직접 할 수 없다면 플랫폼 관리자에게 다음 정보만 요청합니다.

```text
Principal: APIM system-assigned Managed Identity
Scope:     실습에 사용할 Foundry/Azure OpenAI 리소스
Purpose:   OpenAI Responses API inference
Role:      해당 리소스 유형의 모델 호출 역할
```

일반적인 Azure OpenAI 리소스에서는 `Cognitive Services OpenAI User`가 사용되지만, 실제 리소스 유형과 조직 정책에 맞는 최소 역할을 선택합니다.

## 1.5 로컬 도구 확인

```powershell
codex --version
az version
```

Azure CLI는 필수는 아니지만 RBAC와 진단을 반복할 때 유용합니다. Codex가 이미 설치되어 있다는 전제로 이후 모듈을 진행합니다.

## 체크포인트

- [ ] v1 또는 Preview 중 사용할 계약을 선택했다.
- [ ] APIM hostname과 public base path를 정했다.
- [ ] Foundry 모델 배포명을 확인했다.
- [ ] APIM, Foundry, RBAC 작업 권한을 확인했다.
- [ ] secret 값을 문서나 저장소에 적지 않았다.

[🏠 메인](../README.md) | [➡️ 다음: Foundry 모델 준비](../02-foundry-model/)
