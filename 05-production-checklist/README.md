# 모듈 5: Production 전환 체크리스트

[⬅️ 이전: Codex 설정](../04-codex-configuration/) | [🏠 메인](../README.md)

## 학습 목표

- 워크숍 구성을 Production에 적용하기 위한 운영 기준을 정의합니다.
- 회원, credential, trace, 보안, 거버넌스의 담당자와 통제를 결정합니다.
- 단계적 배포, 장애 대응과 비용 관리에 필요한 증적을 준비합니다.

예상 시간은 20~30분입니다. 이 모듈은 Azure 리소스를 추가로 만드는 실습이 아니라, 조직의 Production 승인 기준을 완성하는 의사결정 워크숍입니다.

## 5.1 운영 책임과 산출물

기술적으로 연결되어 있어도 각 통제 항목의 담당자와 승인자가 없으면 Production 준비가 완료된 것이 아닙니다.

| 영역 | 운영 담당 예시 | 승인·협업 주체 | 필수 산출물 |
| --- | --- | --- | --- |
| 회원과 접근 권한 | IAM / 플랫폼 팀 | 개발 조직 관리자 | 가입·변경·탈퇴 절차 |
| APIM과 API product | API 플랫폼 팀 | 서비스 소유자 | API 계약, subscription 정책 |
| Foundry 모델 | AI 플랫폼 팀 | AI 거버넌스 위원회 | 승인 모델·배포 목록 |
| Credential | 보안 / IAM 팀 | 보안 책임자 | 저장·배포·회전 절차 |
| Observability | SRE / 플랫폼 팀 | 개인정보·보안 팀 | 수집 항목, dashboard, alert |
| Incident와 비용 | SRE / FinOps | 서비스 소유자 | Runbook, quota, 예산 경보 |

Production 전환 문서에는 각 영역의 실제 담당 팀, 연락 채널, 승인자를 기록합니다.

## 5.2 회원 체계와 접근 권한

### 사용자와 관리자

- 개인 계정 대신 Microsoft Entra ID 등 조직의 계정 관리 체계를 사용합니다. 해당 인증 체계를 APIM 에 연동시켜서 중앙에서 제어를 구성하실 수 있습니다.
- AD group을 기준으로 `Codex 사용자`, `APIM 운영자`, `Foundry 운영자`, `감사 조회자`를 분리합니다.
- 입사, 부서 이동, 휴직, 퇴사 이벤트가 APIM 접근 권한 회수와 연결되도록 합니다.
- 공유 계정과 공유 관리자 계정을 금지하고 break-glass 계정은 별도로 통제합니다.
- 관리자 역할에는 MFA, Conditional Access, PIM/JIT 승격과 정기 access review를 적용합니다.

### APIM 소비자 식별

공용 subscription 하나를 모든 사용자에게 배포하면 사용자 추적과 선택적 회수가 어렵습니다.

| 방식 | 장점 | 주의점 | 권장 대상 |
| --- | --- | --- | --- |
| 사용자별 subscription | 정확한 추적과 즉시 회수 | 발급·회전 수가 많음 | 소규모 pilot, 고위험 환경 |
| 팀별 subscription | 운영 부담과 추적성의 균형 | 팀 내부 개인 식별이 제한됨 | 일반 개발 팀 |
| Entra OAuth/JWT | key 배포 최소화, 사용자 claim 활용 | client와 APIM OAuth 구성이 필요 | 장기 Production 목표 |
| Workload identity | 비대화형 자동화에 적합 | 사람의 로컬 사용과 분리 필요 | CI/CD, 서비스 계정 |

최소한 팀별로 subscription과 quota를 분리하고, 장기적으로는 Entra 기반 client 인증을 검토합니다. 사람과 자동화가 같은 credential을 사용하지 않게 합니다.

### 체크리스트

- [ ] 접근 대상이 Entra group으로 정의되어 있다.
- [ ] 사용자, APIM 운영자, Foundry 운영자 역할이 분리되어 있다.
- [ ] 가입·변경·탈퇴 시 권한 반영 목표 시간(SLA)이 있다.
- [ ] 퇴사자와 장기 미사용자의 권한을 자동 또는 정기 회수한다.
- [ ] 관리자 권한은 PIM/JIT와 정기 access review 대상이다.
- [ ] 사람과 자동화가 서로 다른 identity와 subscription을 사용한다.

## 5.3 API Key와 Credential 관리

### Credential 경계

```text
Codex ── APIM client credential ──► APIM
APIM  ── Managed Identity ────────► Foundry
```

- 개발자에게 Foundry API key를 배포하지 않습니다.
- APIM은 Foundry backend 호출에 Managed Identity를 사용합니다.
- APIM subscription key를 Git, Markdown, `.env`, `config.toml`, 티켓, 채팅에 저장하지 않습니다.
- Codex의 `env_http_headers`에는 key 값이 아니라 환경 변수 이름만 둡니다.
- 중앙 배포가 필요하면 승인된 secret manager나 endpoint management 도구를 사용합니다.

### Key 수명주기

다음 절차를 문서화하고 실제로 연습합니다.

1. 발급 요청과 승인
2. 사용자 또는 팀 소유자 연결
3. 안전한 전달과 최초 사용 확인
4. primary/secondary key를 이용한 무중단 회전
5. 유출 의심 시 즉시 폐기와 재발급
6. 퇴사·프로젝트 종료 시 회수
7. 만료·회전 이력의 감사 기록

key 값 자체는 관측 로그나 감사 기록에 남기지 않습니다. Production 전환 전 secret scanning으로 저장소와 배포 산출물을 검사합니다.

### 체크리스트

- [ ] Foundry backend 인증이 Managed Identity로 구성되어 있다.
- [ ] APIM client credential마다 소유자와 만료일이 있다.
- [ ] 공유 key의 범위가 최소화되어 있다.
- [ ] primary/secondary key 회전을 검증했다.
- [ ] 유출 대응과 긴급 폐기 Runbook이 있다.
- [ ] 로그에서 subscription key와 Authorization header가 마스킹된다.

## 5.4 Azure Application Insights 기반 Observability

Production에서는 Codex OpenTelemetry와 APIM 진단 로그를 Azure Application Insights에 모아 함께 관찰할 수 있습니다. Codex는 OTLP로 telemetry를 전송하고, OpenTelemetry Collector의 Azure Monitor exporter가 이를 Application Insights로 전달합니다.

```text
Codex CLI ── OTLP/HTTP ──► OpenTelemetry Collector ──► Application Insights
                                ▲                              ▲
                                │                              │
APIM ───── diagnostic settings ─┴──────────────────────────────┘
```

| 계층 | 주요 수집 항목 | 주의할 데이터 |
| --- | --- | --- |
| Codex | API 요청, SSE event, token·지연 metric, tool 승인·실행 결과 | 사용자 prompt, tool output snippet |
| APIM | operation, status, backend status, latency, request ID, token·quota | subscription key, Authorization |
| Foundry | deployment, quota, throttling, 모델 오류 | client credential |

### Application Insights와 Collector 준비

1. Azure Portal에서 workspace-based Application Insights 리소스를 생성합니다.
2. Application Insights의 connection string을 확인하고 공개 저장소에 커밋하지 않습니다.
3. Azure Monitor exporter가 포함된 OpenTelemetry Collector 배포판을 준비합니다.
4. Collector가 수신한 Codex telemetry를 Application Insights로 내보내도록 구성합니다.

최소 Collector 구성 예시는 다음과 같습니다. connection string은 Collector 프로세스의 `APPLICATIONINSIGHTS_CONNECTION_STRING` 환경 변수로 전달합니다.

```yaml
receivers:
  otlp:
    protocols:
      http:
        endpoint: "0.0.0.0:4318"

processors:
  batch: {}

exporters:
  azuremonitor:
    connection_string: ${env:APPLICATIONINSIGHTS_CONNECTION_STRING}

service:
  pipelines:
    logs:
      receivers: [otlp]
      processors: [batch]
      exporters: [azuremonitor]
    metrics:
      receivers: [otlp]
      processors: [batch]
      exporters: [azuremonitor]
```

운영 환경에서는 Collector endpoint에 TLS와 client 인증을 적용하고 외부에 직접 노출하지 않습니다. 필터링, 마스킹 또는 여러 관측 시스템으로의 라우팅도 Collector에서 처리합니다.

### Codex OTel 설정

OpenAI Docs 기준 Codex OTel은 기본 비활성화입니다. 사용자 전역 `%USERPROFILE%\.codex\config.toml`에 다음 설정을 추가합니다.

```toml
[otel]
environment = "production"
log_user_prompt = false

[otel.exporter.otlp-http]
endpoint = "https://YOUR_OTEL_COLLECTOR/v1/logs"
protocol = "binary"
headers = { "Authorization" = "Bearer ${OTLP_TOKEN}" }

[otel.metrics_exporter.otlp-http]
endpoint = "https://YOUR_OTEL_COLLECTOR/v1/metrics"
protocol = "binary"
headers = { "Authorization" = "Bearer ${OTLP_TOKEN}" }
```

`otel`은 project-local `.codex/config.toml`에서 재정의할 수 없는 machine-local 설정입니다. Collector가 내부 신뢰 경계에서 인증 없이 동작하는 실습 환경이라면 `headers`를 생략할 수 있지만, Production에서는 인증된 HTTPS endpoint를 사용합니다.

`log_user_prompt = false`는 사용자 prompt 원문을 가리지만 모든 민감 데이터를 제거하지는 않습니다. `codex.tool_result` event에는 tool output snippet이 포함될 수 있으므로 Collector에서 필요한 event와 필드만 허용하고 Application Insights의 접근 권한과 보존 기간을 민감 데이터 기준으로 설정합니다.

### Application Insights에서 확인

짧은 `codex exec` 요청을 실행한 뒤 수 분 내에 Application Insights의 Logs에서 Codex event와 metric이 수집되는지 확인합니다. APIM도 같은 Application Insights 또는 Log Analytics workspace로 보내면 client와 gateway 관측 데이터를 한곳에서 조회할 수 있습니다. 두 계층을 자동으로 연결해 주는 공통 correlation ID는 별도 설계가 필요하므로, request ID 전파와 로그 필드 매핑을 실제 환경에서 검증합니다.

Dashboard와 alert에는 다음 항목을 우선 포함합니다.

- 전체 요청 수, 성공률, `401`, `403`, `429`, `5xx` 비율
- API 지연시간, time-to-first-token과 SSE 중단
- model·deployment·팀별 token 사용량
- Foundry TPM/RPM quota와 APIM rate limit 사용률
- Collector export 실패와 telemetry 누락

### 체크리스트

- [ ] Application Insights와 OpenTelemetry Collector가 준비되어 있다.
- [ ] Application Insights connection string이 환경 변수 또는 승인된 secret store로 전달된다.
- [ ] Codex OTel 로그와 metric이 Application Insights에 도착한다.
- [ ] APIM 진단 로그를 같은 관측 영역에서 조회할 수 있다.
- [ ] 사용자 prompt 원문은 비활성화되어 있고 tool output snippet의 필터링 정책을 검토했다.
- [ ] 개인정보 마스킹, 보존 기간과 조회 권한이 정의되어 있다.
- [ ] 운영 dashboard와 `401/403/429/5xx`, Collector 실패 alert가 있다.
- [ ] 로그 수집 장애가 모델 요청 장애로 전파되지 않는다.
- [ ] request correlation과 incident 조사 절차를 검증했다.

## 5.5 Security

### 네트워크와 Gateway

- TLS를 강제하고 허용된 APIM hostname만 Codex에 배포합니다.
- 필요한 Responses API operation만 공개하고 불필요한 wildcard route를 피합니다.
- 규제가 엄격하거나 보안 요건상 공용 네트워크를 제한해야 하는 환경에서는 VPN, Azure Private Link(private endpoint), ExpressRoute 등의 네트워크 격리 옵션을 적용할 수 있습니다.
- private endpoint를 사용하면 APIM VNet 통합, private DNS와 outbound 경로를 검증합니다.
- APIM policy에서 client credential을 제거한 뒤 backend로 전달합니다.
- body 변환과 buffering이 SSE streaming을 방해하지 않는지 확인합니다.

### Codex 실행 통제

모델 endpoint 보안과 Codex가 로컬에서 수행하는 tool 실행 보안은 별도 계층입니다.

- 기본 sandbox와 approval policy를 조직 위험 수준에 맞게 정의합니다.
- `danger-full-access`를 일반 사용자 기본값으로 사용하지 않습니다.
- outbound network와 writable root를 필요한 범위로 제한합니다.
- 신뢰되지 않은 저장소의 project config와 instruction을 검토합니다.
- shell에 전달되는 환경 변수를 최소화해 APIM key의 불필요한 전파를 막습니다.
- plugin, MCP server, skill은 승인 목록과 변경 검토 절차를 둡니다.

### 데이터 보호

- source code, prompt, response, tool argument의 데이터 등급을 정합니다.
- 모델 사용이 금지된 repository·경로·데이터 유형을 명시합니다.
- 데이터 저장 위치, 보존 기간, 삭제와 법적 보존 요구사항을 검토합니다.
- content safety 또는 DLP 검사가 필요한 요청 유형을 정합니다.

### 체크리스트

- [ ] 위협 모델에 client, APIM, Foundry, 로컬 tool 실행이 포함되어 있다.
- [ ] least privilege, MFA와 PIM/JIT가 적용되어 있다.
- [ ] network 및 data exfiltration 경로를 검토했다.
- [ ] 규제·보안 요건에 따라 VPN, Private Link, ExpressRoute 등 필요한 네트워크 격리 방식을 검토하고 적용 여부를 결정했다.
- [ ] sandbox, approval, network access의 조직 기본값이 있다.
- [ ] 허용된 plugin/MCP/skill 목록과 검토 절차가 있다.
- [ ] 침해 시 key 폐기, 접근 차단과 로그 보존 절차가 있다.

## 5.6 Governance와 변경 관리

### 승인 카탈로그

| 항목 | 기록할 내용 |
| --- | --- |
| Model | 승인 모델, 버전, 지역, 사용 목적, 소유자 |
| Deployment | deployment name, quota, 환경, backend endpoint |
| APIM API | public base URL, operation, product, policy 버전 |
| Codex config | provider ID, profile, 배포 방법, 기준 버전 |
| Data | 허용·금지 등급, residency, retention |
| Integration | 승인된 MCP, plugin, skill, 외부 endpoint |

### 변경과 사용 정책

- model 또는 API version은 dev → pilot → production 순으로 승격합니다.
- APIM policy와 Codex 표준 설정은 코드로 관리하고 peer review를 거칩니다.
- 변경 전 API 계약, tool calling, streaming, latency와 token 사용량을 회귀 테스트합니다.
- deployment와 endpoint 전환에는 rollback 절차를 둡니다.
- 허용된 사용 사례와 금지된 데이터 입력을 문서화합니다.
- 생성 코드의 검토, 테스트, 라이선스와 보안 책임을 사용자에게 교육합니다.
- 정책 예외에는 소유자, 사유, 승인자와 만료일을 둡니다.

### 체크리스트

- [ ] 승인 모델·배포·API·통합 카탈로그가 있다.
- [ ] 모델과 API version 변경에 회귀 테스트와 rollback이 있다.
- [ ] APIM policy와 Codex 기준 설정이 형상 관리된다.
- [ ] 데이터 등급별 허용·금지 사용 정책이 있다.
- [ ] 예외 승인에는 소유자, 사유와 만료일이 있다.
- [ ] 사용자 교육과 정책 확인 절차가 있다.

## 5.7 안정성, 용량과 비용

- APIM capacity와 Foundry TPM/RPM quota를 예상 동시 사용자 기준으로 산정합니다.
- 팀 또는 subscription별 rate limit과 quota로 noisy neighbor를 방지합니다.
- `429`의 `Retry-After`를 존중하고 client/APIM retry 증폭을 점검합니다.
- 장애 시 사용할 보조 deployment, region 또는 수동 전환 절차를 정합니다.
- 장시간 SSE 연결의 APIM, load balancer와 firewall timeout을 검증합니다.
- 팀·deployment별 token 비용을 태깅하고 예산 임계값을 alert로 연결합니다.
- 사용하지 않는 subscription, deployment와 quota를 정기적으로 회수합니다.

## 5.8 Production 승인 테스트

Production credential로 전환하기 전에 비운영 데이터로 검증합니다.

1. APIM REST smoke test와 Codex `exec`가 성공한다.
2. streaming 응답이 중단되거나 buffering되지 않는다.
3. `/status`에서 승인된 provider와 deployment가 선택된다.
4. 같은 요청을 Codex OTel과 APIM trace에서 연결할 수 있다.
5. 잘못된 key, 권한 없는 사용자와 과도한 요청이 각각 차단된다.
6. key 회전 후 중단 없이 새 credential로 전환된다.
7. quota 초과와 backend 장애의 alert와 Runbook이 동작한다.
8. 이전 model, APIM policy와 Codex config로 rollback할 수 있다.

## 5.9 최종 Go-Live 체크리스트

- [ ] 회원·권한: 그룹, 역할, onboarding/offboarding과 비상 접근이 승인되었다.
- [ ] Credential: client/backend 인증이 분리되고 회전·폐기 절차가 검증되었다.
- [ ] Observability: correlation, dashboard, alert와 보존 정책이 적용되었다.
- [ ] Privacy: prompt와 secret이 기본 로그에서 제외된다.
- [ ] Security: network, sandbox, approval과 incident 대응 검토가 끝났다.
- [ ] Governance: 승인 카탈로그, 사용 정책, 변경·예외 절차가 있다.
- [ ] Reliability: quota, rate limit, streaming과 장애 전환을 검증했다.
- [ ] FinOps: 비용 소유자, budget alert와 정기 사용량 검토가 있다.

모든 필수 항목에 담당자, 완료일, 증적 링크를 연결한 후 Production 전환을 승인합니다.

## 참고 자료

- [OpenAI Docs: Codex advanced configuration](https://developers.openai.com/codex/config-advanced)
- [OpenAI Docs: Agent approvals and security](https://developers.openai.com/codex/agent-approvals-security)
- [Microsoft Learn: AI gateway capabilities in APIM](https://learn.microsoft.com/azure/api-management/genai-gateway-capabilities)
- [Microsoft Learn: Monitor Azure API Management](https://learn.microsoft.com/azure/api-management/api-management-howto-use-azure-monitor)
- [Microsoft Learn: Application Insights overview](https://learn.microsoft.com/azure/azure-monitor/app/app-insights-overview)
- [참고 워크숍: GitHub Copilot Observability](https://github.com/MSFT-AI-Build/github-copilot-ops/tree/main/04-observability)

[⬅️ 이전: Codex 설정](../04-codex-configuration/) | [🏠 메인](../README.md)
