---
layout: default
title: NLB mutual TLS 테스트 보고서
---

# NLB mutual TLS 테스트 보고서

- 대상 기능: Network Load Balancer mutual TLS (2026-11-19 발표)
- 리전: ap-northeast-2 (서울)
- 테스트 일자: 2026-11-27
- 상태: 완료

## 목차

1. [요약](#summary)
2. [배경과 목적](#background)
3. [테스트 환경](#environment)
4. [테스트 케이스와 결과](#test-cases)
5. [실측 결과](#measurements)
6. [문서와 다른 점](#doc-discrepancies)
7. [ALB mTLS와 비교](#alb-comparison)
8. [한계와 범위 밖](#limitations)
9. [비용과 정리 결과](#cost-cleanup)
10. [참고 문서](#references)

## 요약
{: #summary}

- 전체 결과: 필수 시나리오 7개(하위 케이스 12개)가 모두 기대대로 동작했습니다. 새로 배포한 환경에서 3회 연속 같은 결과가 나왔습니다.
- 시나리오 7개 통과 여부(3회 반복): 12/12 통과, 3회 반복 모두 같은 결과. 판정한 연결 48개 모두 access log 행과 짝지어짐
- 문서와 다르게 동작한 점: 인증서 없이 접속하면 문서 표에 없는 `Failed:ClientCertMissing`이 기록됩니다. CloudFormation 스택은 검증 메시지를 남긴 채 완료되고, identity filter는 적용되지 않습니다. 거부 행에는 인증서 필드가 남지 않습니다.
- 운영 시 주의할 점: 이번 테스트에서 설정 변경(CRL, 만료 허용, CA 번들 교체)은 API 응답 후 약 45~60초 뒤에 처음 반영된 것이 관측됐습니다(보장된 시간은 아님). 테스트한 환경에서는 filter를 CloudFormation이 아니라 CLI/API로 설정해야 했습니다. 리스너 삭제 후 trust store 연결이 2분 넘게 남아 바로 삭제되지 않을 수 있습니다.

## 배경과 목적
{: #background}

NLB는 이제 TLS 리스너에서 클라이언트 인증서를 직접 검증합니다(`verify` 모드). 이 보고서는 공식 문서에 적힌 동작이 실제로 그대로 나오는지, 문서에 없는 동작은 어떻게 나오는지를 직접 배포해 확인한 기록입니다.

- 필수 시나리오 7개를 클라이언트 증상과 access log 사유 코드로 함께 판정합니다.
- 각 시나리오는 새로 배포한 환경에서 3회 반복해 같은 결과가 나와야 통과로 봅니다.
- 문서로 확인되지 않은 동작(메트릭 이름, 세션 재개, 거부 증상 등)은 실측 항목으로 따로 기록합니다.

문서로 확인한 기능 요점:

| 항목 | 내용 |
|---|---|
| 모드 | NLB는 `off`, `verify`만 지원 (`passthrough` 없음) |
| 리스너 | TLS 리스너. LB당 verify 리스너 최대 2개, 조정 불가 |
| trust store | CA 번들과 CRL(PEM)을 S3에서 import. 리스너 하나에 trust store 하나 |
| 검증 순서 | 체인 검증 → client certificate identity filter → CRL |
| identity filter | `subject-cn`, `subject-ou`, `san-dns`, `san-uri`. `exact`/`prefix`, OR 결합, 리스너당 최대 100개 |
| 세션 재개 | verify 모드에서 지원하지 않음 |
| 관측 | access log의 `tls_verify_status`, `leaf_client_cert_*` 필드 |

## 테스트 환경
{: #environment}

![테스트 환경 구성도]({{ "/assets/test-architecture.svg" | relative_url }})

- 그림의 번호는 테스트 트래픽이 지나는 경로입니다.
- IGW는 EC2 패키지 설치와 Session Manager 접속용이고, 테스트 트래픽은 VPC 안에서만 흐릅니다.

| 구성 요소 | 내용 |
|---|---|
| VPC | 퍼블릭 서브넷 2개 (AZ a, c) |
| 타깃 EC2 | t4g.small 1대. 포트 8080에서 받은 요청 헤더를 그대로 돌려주는 HTTP 서버 |
| 클라이언트 EC2 | t4g.small 1대. `curl`, `openssl` 3.x. Session Manager로 접속 |
| NLB | internal, 서브넷 2개. 타깃 그룹 TCP 8080, instance 타깃. 타깃이 AZ a에만 있어 교차 영역 부하 분산을 켬 |
| 서버 인증서 | 자체 CA로 `mtls.lab.example` 인증서를 발급해 ACM에 import |
| trust store | CA 번들 = CA-A. 두 리스너가 함께 사용 |
| 리스너 443 | TLS, `ELBSecurityPolicy-TLS13-1-2-2021-06`, `Mode=verify`, filter 없음 |
| 리스너 8443 | 443과 같은 설정에 filter `san-uri` / `prefix` / `spiffe://lab.example/workload/app` 추가 |
| access log | S3로 전송 (5분 주기, best-effort) |
| 배포 | 기반 인프라는 CloudFormation, mTLS 설정은 AWS CLI |

도구 버전:

| 도구 | 버전 |
|---|---|
| AWS CLI | aws-cli 1.45.64 + botocore 1.44.12 (filter 파라미터를 지원하는 SDK 모델 필요) |
| curl | 8.21.0 (클라이언트 EC2, Amazon Linux 2023 arm64) |
| OpenSSL | 3.5.8 |

### 인증서

모두 OpenSSL 자체 CA로 만들었습니다. 키는 모두 ECDSA P-256, 클라이언트 인증서에는 EKU `clientAuth`와 KU `digitalSignature`를 넣었고 체인은 root → leaf입니다. CRL은 PEM 형식입니다.

| ID | 발급 CA | 특징 |
|---|---|---|
| `good` | CA-A | SAN URI `spiffe://lab.example/workload/app` |
| `untrusted` | CA-B (trust store에 없음) | SAN URI는 `good`과 같음 |
| `expired` | CA-A | 이미 지난 유효기간 |
| `revoked` | CA-A | CRL에 serial 등록 |
| `other` | CA-A | SAN URI `spiffe://lab.example/workload/batch` |
| `nosan` | CA-A | SAN 없음, CN만 있음 |
| `new` | CA-C (교체용) | SAN URI는 `good`과 같음 |

## 테스트 케이스와 결과
{: #test-cases}

판정 기준: 클라이언트 결과와 access log의 `tls_verify_status`가 모두 기대 결과와 같으면 통과입니다. 새로 배포한 환경에서 3회 반복했습니다.

| TC | 시나리오 | 기대 결과 | 관측된 `tls_verify_status` | 1회차 | 2회차 | 3회차 |
|---|---|---|---|---|---|---|
| TC-01 | 443에 `good` | HTTP 200, `Success` | `Success` | ✅ PASS | ✅ PASS | ✅ PASS |
| TC-02 | 443에 인증서 없이 | 연결 거부. 코드는 문서에 없어 실측(M3) | `Failed:ClientCertMissing` | ✅ PASS | ✅ PASS | ✅ PASS |
| TC-03 | 443에 `untrusted` | 거부, `Failed:ClientCertUntrusted` | `Failed:ClientCertUntrusted` | ✅ PASS | ✅ PASS | ✅ PASS |
| TC-04a | 443에 `expired` | 거부, `Failed:ClientCertExpired` | `Failed:ClientCertExpired` | ✅ PASS | ✅ PASS | ✅ PASS |
| TC-04b | `IgnoreClientCertificateExpiry=true`에서 `expired` | HTTP 200, `Success` | `Success` | ✅ PASS | ✅ PASS | ✅ PASS |
| TC-05 | CRL 추가 후 `revoked`, 이어서 `good` | `revoked` 거부 `Failed:ClientCertCrlHit`, `good`은 `Success` | `Failed:ClientCertCrlHit` / `Success` | ✅ PASS | ✅ PASS | ✅ PASS |
| TC-06a | 8443에 `good` (filter 일치) | HTTP 200, `Success` | `Success` | ✅ PASS | ✅ PASS | ✅ PASS |
| TC-06b | 8443에 `other`, `nosan` (불일치) | 거부, `Failed:CertificateFilterNoMatch` | `Failed:CertificateFilterNoMatch` | ✅ PASS | ✅ PASS | ✅ PASS |
| TC-06c | 443에 `other` (filter는 리스너 단위) | HTTP 200, `Success` | `Success` | ✅ PASS | ✅ PASS | ✅ PASS |
| TC-06d | 8443에 `untrusted` (평가 순서) | 거부, `Failed:ClientCertUntrusted`. 문서의 검증 순서와 일치하는 결과 (단, `untrusted`도 filter에 맞는 SAN URI라 순서 자체를 증명하지는 않음) | `Failed:ClientCertUntrusted` | ✅ PASS | ✅ PASS | ✅ PASS |
| TC-07a | CA-A+CA-C 번들로 교체 후 `good`, `new` | 둘 다 HTTP 200, `Success` | `Success` / `Success` | ✅ PASS | ✅ PASS | ✅ PASS |
| TC-07b | CA-C만 든 번들로 교체 후 `good`, `new` | `good` 거부 `Failed:ClientCertUntrusted`, `new`는 `Success` | `Failed:ClientCertUntrusted` / `Success` | ✅ PASS | ✅ PASS | ✅ PASS |

- CA 교체(TC-07)는 개별 CA를 넣고 빼는 API가 없어서 `modify-trust-store`로 번들 전체를 두 번 교체하는 방식으로 했습니다.
- 관찰 메모: TC-02의 기대 코드는 1회차 관찰 결과(`Failed:ClientCertMissing`)로 정한 뒤 3회 반복했습니다. 설정을 바꾸는 TC(04b, 05, 07)는 회차마다 원래 상태로 되돌렸습니다. 판정 로직이 항상 통과만 내지 않는지 보려고 trust store에 CA-B를 일부러 넣고 TC-03을 실행했더니 `untrusted`가 통과했고, 실행기는 FAIL로 판정했습니다.

## 실측 결과
{: #measurements}

공식 문서에 없거나 블로그에만 있는 동작을 직접 측정한 결과입니다.

| ID | 확인한 것 | 문서 상태 | 결과 |
|---|---|---|---|
| M1 | mTLS CloudWatch 메트릭 이름 (`AWS/NetworkELB`) | 메트릭 문서에는 TLS 메트릭이 `ClientTLSNegotiationErrorCount`, `TargetTLSNegotiationErrorCount`뿐. mTLS 메트릭이 있다는 설명은 블로그 출처 | 테스트한 NLB의 약 3시간 범위에서 mTLS 전용 메트릭이나 trust store·리스너 차원은 관측되지 않았습니다. 거부된 연결은 `ClientTLSNegotiationErrorCount`에 집계됐고, 클라이언트가 reset을 받았는데도 `TCP_ELB_Reset_Count`는 0이었습니다. |
| M2 | verify 모드에서 세션 재개 시도 시 동작 (TLS 1.2 / 1.3) | "지원하지 않음"이라고만 적혀 있음 | TLS 1.2 `-reconnect`, TLS 1.2/1.3 세션 재사용 모두 오류 없이 full handshake(`New`)로 처리되고 요청은 성공했습니다. 다만 mTLS가 없는 비교용 리스너에서도 재개되지 않아, mTLS 때문인지는 구분하지 못했습니다. |
| M3 | 인증서 없이 접속할 때의 사유 코드 | 사유 코드 표에 해당 항목 없음 | `Failed:ClientCertMissing` (문서의 사유 코드 표에 없음). leaf 필드는 모두 `-` |
| M4 | 거부 시 클라이언트 증상 (curl 종료 코드, 오류 시점). 인증서 없음과 신뢰하지 않는 CA 두 경우를 측정 | connection reset, TLS 1.3은 첫 요청 시점에 reset (블로그 출처) | HTTP 응답은 없습니다. TLS 1.2: 핸드셰이크 도중 실패, curl `exit 35` (`wrong version number`). TLS 1.3: 클라이언트 쪽 핸드셰이크는 끝나고 첫 요청 시점에 실패, curl `exit 56` (`Connection reset by peer`, openssl은 `bad record type`). 블로그의 curl 코드(인증서 없음 55, 신뢰하지 않는 CA 56)와는 다르게 TLS 버전에 따라 갈렸습니다. |
| M5 | 클라이언트 인증서 정보가 타깃에 전달되는지 | 전달되지 않음 (블로그 출처) | TCP 타깃 그룹, Proxy Protocol v2 꺼짐(기본) 구성에서 타깃이 받은 헤더(`Host`, `User-Agent`, `Accept`, 테스트용 `X-Lab-Probe`)와 타깃 로그에 인증서 정보가 없었습니다. 타깃이 본 peer 주소는 클라이언트 IP였습니다. |
| M6 | CA 번들 교체 반영 시간, 이미 열린 연결에 미치는 영향 | 문서에 없음 | 한 번 측정한 결과입니다. CA-A+C → C 교체 후 기존 인증서의 신규 연결은 52초 뒤 처음 거부됐습니다. 교체 전에 열어 둔 연결은 62초 뒤에도 요청이 성공했습니다(그 이상은 측정하지 않음). 다른 설정 변경도 45~60초 뒤에 반영됐습니다. |
| M7 | CloudFormation으로 `ClientCertificateIdentityFilters` 설정 | 사용자 가이드와 CFN 레퍼런스가 다름 (D1) | 스택은 `CREATE_COMPLETE`가 되지만 filter는 적용되지 않습니다(실패로 처리되지 않음). 스택 이벤트에 `extraneous key [ClientCertificateIdentityFilters] is not permitted`가 남고, 리스너의 filter는 `[]`입니다. cfn-lint도 이 속성을 거부합니다. filter는 CLI/API로 설정해야 합니다. |
| M8 | `modify-listener`에서 filter를 빼고 보내면 기존 filter가 유지되는지 | "지정하지 않은 속성은 유지"라고만 적혀 있음 | 유지됩니다. filter를 빼고 `--mutual-authentication`을 보내도 기존 filter는 그대로였습니다. 지우려면 `ClientCertificateIdentityFilters: []`를 명시해야 합니다. |

## 문서와 다른 점
{: #doc-discrepancies}

문서끼리 서로 다르게 적혀 있어 테스트로 확인한 항목입니다.

| # | 내용 | 문서 A | 문서 B | 확인 결과 |
|---|---|---|---|---|
| D1 | CFN에서 identity filter 지원 | 사용자 가이드: `MutualAuthentication`에 `ClientCertificateIdentityFilters` 목록을 넣는 예시가 있음 | CFN 레퍼런스: `MutualAuthentication` 속성에 filter가 없음 | CFN 레퍼런스 쪽이 맞았습니다. 2026-11-27 기준 서울 리전에서 CloudFormation은 filter를 적용하지 않습니다(M7). |
| D8 | mTLS 전용 CloudWatch 메트릭 | 블로그 출처: handshake 수, 오류 수, 지연, 인증서 만료 메트릭을 게시한다고 함 | 메트릭 문서: mTLS 메트릭 없음 | 테스트한 범위에서는 메트릭 문서와 같았습니다. mTLS 전용 메트릭이나 trust store·리스너 차원은 관측되지 않았습니다(M1). |
| D10 | 거부된 연결의 `tls_handshake_time`·`tls_cipher` | 필드 정의: 핸드셰이크 실패 시 `-` | 같은 문서의 `Failed:ClientCertUntrusted` 예시에는 값이 채워져 있음 | 필드 정의 쪽이 맞았습니다. 모든 거부 행에서 `tls_handshake_time`, `tls_cipher`, `tls_protocol_version`이 `-`였습니다. |
| D11 | 거부된 연결의 `leaf_client_cert_*` 값 | 필드 정의: 인증서를 읽지 못할 때만 `-`. 문서 예시의 `ClientCertUntrusted` 행에 subject·serial이 있음 | 블로그 출처: 신뢰하지 않는 인증서의 필드는 보지 않는다고 함 | 모든 거부 행에서 leaf subject·validity·serial이 `-`였습니다. 문서 예시와 다르고, 거부된 연결은 인증서가 아니라 클라이언트 주소·포트와 시각으로만 찾을 수 있습니다. |
| D15 | trust store 시간당 요금 | trust store 공유 문서: trust store 연결마다 시간 단위로 과금 | 요금 페이지 본문에는 금액이 없음 | 이번 테스트로는 확인하지 않았습니다. 실제 과금은 Cost Explorer 반영 후 확인 예정입니다. |

## ALB mTLS와 비교
{: #alb-comparison}

문서로 확인한 사실만 적었습니다. ALB는 이번에 배포하지 않았습니다.

| 항목 | NLB | ALB |
|---|---|---|
| `Mode` 허용값 | `off`, `verify` | `off`, `passthrough`, `verify` |
| 적용 리스너 | TLS 리스너 | HTTPS 리스너 |
| client certificate identity filter | 지원 (`ClientCertificateIdentityFilters`, NLB 전용) | 해당 속성 없음 |
| LB당 verify 리스너 | 최대 2개, 조정 불가 | 이 보고서에서 확인하지 않음 |
| 타깃에 인증서 정보 전달 | 전달하지 않음 (블로그 출처). 실측: TCP 타깃 그룹·Proxy Protocol v2 꺼짐 구성에서 타깃에 인증서 정보 없음(M5) | 이 보고서에서 확인하지 않음 |
| trust store | 같은 elbv2 trust store 리소스 | 같음 |
| 같은 trust store를 ALB·NLB가 함께 쓰기 | 문서에 없음. 시험하지 않음 | — |

## 한계와 범위 밖
{: #limitations}

- 한 리전(ap-northeast-2), 한 계정, OpenSSL 자체 CA에서만 확인했습니다. ACM Private CA는 쓰지 않았습니다.
- 클라이언트 증상(curl 오류 코드와 메시지)은 curl/OpenSSL 버전에 따라 다를 수 있습니다. 판정은 access log 사유 코드를 기준으로 했습니다.
- access log는 5분 주기, best-effort로 게시되어 일부 연결이 빠질 수 있습니다.
- `ClientCertificateIdentityFilters`는 이 파라미터를 지원하는 최신 AWS CLI/SDK가 필요합니다.

시험하지 않은 것:

- RAM으로 다른 계정과 trust store 공유
- EKS / AWS Load Balancer Controller 연동
- ALB mTLS 실배포 (위 비교표로만 설명)
- MQTT 같은 비HTTP 프로토콜
- OCSP (API의 `RevocationType` 값은 `CRL`뿐)
- `IgnoreClientCertificateExpiry`가 중간·루트 CA 만료에도 적용되는지
- CRL `nextUpdate`가 지났을 때의 동작, `ClientCertCrlProcessingError`가 나는 조건
- 체인 깊이·크기 같은 quota 경계값
- 같은 trust store를 ALB와 NLB가 함께 쓰는 구성

## 비용과 정리 결과
{: #cost-cleanup}

요금 체계 (문서·요금 페이지 기준):

- trust store는 NLB와의 연결(association)마다 시간 단위로 과금됩니다.
- mTLS 연결은 NLCU 1개당 신규 연결 25/s로 계산됩니다 (일반 TLS는 50/s).

| 항목 | 결과 |
|---|---|
| 테스트 전체 비용 (USD) | 약 0.2 미만 (추정: 환경 2개 합계 약 1.5시간 + 검증용 NLB 약 5분, 환경당 시간당 약 0.09 기준) |
| trust store 실제 과금 (M9) | 확인 전 (Cost Explorer 반영 대기) |
| 정리 후 남은 리소스 수 (태그 기준) | 0개. 태그 조회 결과에는 이미 종료된 EC2·삭제된 VPC가 지연 반영으로 남아 보였고, 각 서비스 API로 실제로 없는 것을 확인했습니다 |

정리 순서: 리스너 삭제 → trust store 연결이 비워질 때까지 대기 → `delete-trust-store` → ACM 서버 인증서 삭제 → CloudFormation 스택 삭제 → S3 버킷 비우기와 삭제. 리스너에 연결된 trust store는 삭제할 수 없습니다. 실측에서는 리스너를 지운 뒤에도 연결이 2분 넘게 남아 `TrustStoreInUse`가 났으므로, 기다렸다가 다시 시도해야 합니다.

## 참고 문서
{: #references}

- [Mutual TLS for Network Load Balancers](https://docs.aws.amazon.com/elasticloadbalancing/latest/network/mutual-authentication.html)
- [Configure mutual TLS](https://docs.aws.amazon.com/elasticloadbalancing/latest/network/configure-mutual-authentication.html)
- [Client certificate identity filters](https://docs.aws.amazon.com/elasticloadbalancing/latest/network/client-certificate-filters.html)
- [Access logs for your Network Load Balancer](https://docs.aws.amazon.com/elasticloadbalancing/latest/network/load-balancer-access-logs.html)
- [CloudWatch metrics for your Network Load Balancer](https://docs.aws.amazon.com/elasticloadbalancing/latest/network/load-balancer-cloudwatch-metrics.html)
- [Quotas for your Network Load Balancers](https://docs.aws.amazon.com/elasticloadbalancing/latest/network/load-balancer-limits.html)
- [Trust store sharing](https://docs.aws.amazon.com/elasticloadbalancing/latest/network/mutual-authentication-trust-store-sharing.html)
- [API: MutualAuthenticationAttributes](https://docs.aws.amazon.com/elasticloadbalancing/latest/APIReference/API_MutualAuthenticationAttributes.html)
- [API: ClientCertificateIdentityFilter](https://docs.aws.amazon.com/elasticloadbalancing/latest/APIReference/API_ClientCertificateIdentityFilter.html)
- [API: ModifyTrustStore](https://docs.aws.amazon.com/elasticloadbalancing/latest/APIReference/API_ModifyTrustStore.html)
- [API: RevocationContent](https://docs.aws.amazon.com/elasticloadbalancing/latest/APIReference/API_RevocationContent.html)
- [AWS CLI: elbv2 modify-listener](https://docs.aws.amazon.com/cli/latest/reference/elbv2/modify-listener.html)
- [CloudFormation: Listener MutualAuthentication](https://docs.aws.amazon.com/AWSCloudFormation/latest/TemplateReference/aws-properties-elasticloadbalancingv2-listener-mutualauthentication.html)
- [CloudFormation: AWS::ElasticLoadBalancingV2::TrustStore](https://docs.aws.amazon.com/AWSCloudFormation/latest/TemplateReference/aws-resource-elasticloadbalancingv2-truststore.html)

"블로그 출처"로 표시한 내용은 AWS Networking & Content Delivery 블로그의 "Introducing mTLS for Network Load Balancer" 글에서 가져왔고, 사용자 가이드에는 없는 내용입니다.
