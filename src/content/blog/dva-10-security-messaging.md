---
title: "[DVA-C02 #10] 보안과 메시징 - KMS, Cognito, SQS, Kinesis"
description: "AWS DVA-C02 스터디 마지막 편. 암호화(KMS), 인증(Cognito, STS), 메시징(SQS, SNS, Kinesis)을 정리합니다."
publishedAt: 2025-05-16
category: "cloud"
tags: ["AWS", "DVA-C02", "KMS", "Cognito", "SQS", "Kinesis"]
---

> AWS DVA-C02 스터디 시리즈 마지막 편입니다. [9편: 모니터링](/blog/dva-09-monitoring)

## KMS (Key Management Service)

AWS의 암호화 키 관리 서비스다. S3, EBS, RDS 등 대부분의 AWS 서비스에서 암호화에 KMS 키를 사용한다.

### 암호화 유형

| 유형 | 설명 |
|------|------|
| 전송 중 암호화 (In-Transit) | HTTPS/TLS로 데이터 이동 중 보호 |
| 저장 시 암호화 (At-Rest) | 저장된 데이터를 암호화 |
| 클라이언트 사이드 암호화 | 클라이언트에서 직접 암호화 후 전송 |

### KMS 키 유형

- **AWS Managed Key**: AWS가 자동으로 관리. 서비스별로 생성 (aws/s3, aws/rds 등)
- **Customer Managed Key (CMK)**: 사용자가 직접 생성·관리. 키 정책, 회전 설정 가능
- **Customer Provided Key**: 사용자가 외부에서 가져온 키

### Envelope Encryption

4KB보다 큰 데이터를 암호화할 때 사용하는 방식이다.

```
1. KMS에 Data Key 생성 요청 (GenerateDataKey API)
2. 평문 Data Key + 암호화된 Data Key를 받음
3. 평문 Data Key로 데이터를 암호화
4. 암호화된 Data Key를 데이터와 함께 저장
5. 평문 Data Key는 메모리에서 삭제
```

복호화 시에는 암호화된 Data Key를 KMS에 보내 복호화한 뒤, 그 키로 데이터를 복호화한다.

### API 요청 제한

KMS에는 초당 요청 수 제한이 있다. 제한에 걸리면 `ThrottlingException`이 발생한다.

완화 방법:
- **지수 백오프**로 재시도
- **Data Key Caching**: GenerateDataKey 결과를 캐싱하여 KMS 호출 횟수를 줄임
- **Service Quota** 증가 요청

## STS (Security Token Service)

임시 보안 자격 증명을 발급하는 서비스다.

### 주요 API

| API | 용도 |
|-----|------|
| `AssumeRole` | 다른 역할의 임시 자격 증명 발급 |
| `GetSessionToken` | MFA 기반 임시 자격 증명 |
| `GetCallerIdentity` | 현재 호출자의 계정/역할 정보 확인 |
| `AssumeRoleWithSAML` | SAML 기반 페더레이션 |
| `AssumeRoleWithWebIdentity` | 웹 ID 기반 페더레이션 (Cognito 권장) |

### Cross-Account Access

다른 AWS 계정의 리소스에 접근할 때 `AssumeRole`을 사용한다. 대상 계정에서 역할을 만들고, 소스 계정에서 해당 역할을 Assume하는 방식이다.

## Cognito

사용자 인증과 권한 부여를 제공하는 서비스다.

### User Pool

사용자 등록, 로그인, 비밀번호 관리를 처리한다. JWT 토큰을 발급한다.

- 이메일/비밀번호 로그인
- 소셜 로그인 (Google, Facebook, Apple)
- MFA 지원
- **Lambda 트리거**: 회원가입 전/후, 인증 전/후에 커스텀 로직 실행

API Gateway, ALB와 연동하여 인증된 사용자만 API에 접근하도록 할 수 있다.

### Identity Pool

인증된 사용자에게 **AWS 리소스에 대한 임시 자격 증명**을 발급한다. User Pool에서 인증 → Identity Pool에서 AWS 자격 증명 획득 → S3, DynamoDB 등 직접 접근하는 흐름이다.

비인증 사용자(게스트)에게도 제한된 권한을 줄 수 있다.

### User Pool vs Identity Pool

| 구분 | User Pool | Identity Pool |
|------|-----------|---------------|
| 역할 | 인증 (Authentication) | 인가 (Authorization) |
| 반환 | JWT 토큰 | AWS 임시 자격 증명 |
| 사용처 | API Gateway, ALB 인증 | AWS 서비스 직접 접근 |

보통 User Pool → Identity Pool 순서로 함께 사용한다.

## SQS (Simple Queue Service)

완전 관리형 메시지 큐 서비스다. 서비스 간 비동기 통신에 사용한다.

### Standard vs FIFO

| 특성 | Standard | FIFO |
|------|----------|------|
| 순서 | 보장 안 됨 (최선 노력) | 엄격한 순서 보장 |
| 중복 | 최소 1회 전달 (중복 가능) | 정확히 1회 전달 |
| 처리량 | 무제한 | 초당 300건 (배치 시 3,000) |
| 이름 | 자유 | `.fifo` 접미사 필수 |

### 주요 개념

**Visibility Timeout**: 메시지를 받은 후 다른 소비자에게 보이지 않는 시간. 이 시간 안에 처리를 완료하고 삭제해야 한다. 기본 30초, 최대 12시간.

**Dead Letter Queue (DLQ)**: 지정된 횟수(MaxReceiveCount)만큼 처리에 실패한 메시지가 이동하는 큐. 디버깅과 분석에 사용한다.

**Long Polling**: 메시지가 없을 때 즉시 빈 응답을 반환하지 않고, 메시지가 도착할 때까지 최대 20초 대기한다. 불필요한 API 호출을 줄여 비용을 절감한다.

**Extended Client Library**: 256KB 이상의 대용량 메시지를 보내야 할 때, 실제 데이터는 S3에 저장하고 SQS에는 참조 정보만 보낸다.

### Lambda와 SQS

Lambda를 SQS 트리거로 설정하면, Lambda가 큐를 폴링하여 메시지를 배치로 처리한다. 처리 실패 시 Visibility Timeout 후 재처리된다.

## SNS (Simple Notification Service)

Pub/Sub 메시징 서비스다. 하나의 메시지를 여러 구독자에게 동시에 전달한다.

- **구독자**: SQS, Lambda, HTTP 엔드포인트, 이메일, SMS
- **Fan-out 패턴**: SNS + SQS 조합으로 하나의 이벤트를 여러 큐에 동시 전달
- **Message Filtering**: 구독자별로 필터 정책을 설정하여 특정 메시지만 받을 수 있음

### SNS + SQS Fan-out

```
Producer → SNS Topic → SQS Queue A (주문 처리)
                      → SQS Queue B (알림 전송)
                      → SQS Queue C (분석 저장)
```

하나의 이벤트를 여러 서비스가 독립적으로 처리해야 할 때 자주 사용하는 패턴이다.

## Kinesis

실시간 스트리밍 데이터를 수집하고 처리하는 서비스다.

### Kinesis Data Streams

실시간으로 대량의 데이터를 수집한다.

- **Shard**: 데이터 처리 단위. 샤드당 읽기 2MB/s, 쓰기 1MB/s.
- **보존 기간**: 1일(기본) ~ 365일
- 데이터가 들어가면 **삭제할 수 없다** (불변)

**용량 모드**:
- Provisioned: 샤드 수를 직접 관리
- On-Demand: 자동 스케일링 (최근 30일 피크 기준)

### Kinesis Data Firehose

데이터를 변환 없이 또는 간단한 변환 후 **목적지로 전달**하는 서비스다. S3, Redshift, OpenSearch, HTTP 엔드포인트로 보낼 수 있다. 완전 관리형이라 샤드 관리가 필요 없다.

### SQS vs SNS vs Kinesis

| 특성 | SQS | SNS | Kinesis |
|------|-----|-----|---------|
| 모델 | 큐 (1:1) | Pub/Sub (1:N) | 스트림 |
| 소비 | 소비자가 풀링 | 구독자에게 푸시 | 소비자가 풀링 |
| 데이터 보존 | 최대 14일 | 보존 없음 | 최대 365일 |
| 순서 | FIFO만 보장 | FIFO만 보장 | 샤드 내 보장 |
| 리플레이 | 불가 (삭제됨) | 불가 | 가능 |
| 사용 사례 | 비동기 작업 큐 | 알림/팬아웃 | 실시간 분석, 로그 |

## Step Functions

여러 AWS 서비스를 워크플로우로 조합하는 서버리스 오케스트레이션 서비스다.

### 상태 유형

| 상태 | 역할 |
|------|------|
| Task | Lambda, ECS 등 실행 |
| Choice | 조건 분기 |
| Parallel | 병렬 실행 |
| Wait | 대기 |
| Map | 배열 항목별 반복 처리 |
| Pass | 데이터 전달/변환 |
| Succeed/Fail | 종료 |

### Standard vs Express

| 구분 | Standard | Express |
|------|----------|---------|
| 실행 시간 | 최대 1년 | 최대 5분 |
| 가격 | 상태 전환당 과금 | 실행 수/시간당 과금 |
| 실행 보장 | 정확히 1회 | 최소 1회 |
| 사용 사례 | 장기 실행 워크플로우 | 대량 데이터 처리, IoT |

### 에러 처리

`Retry`와 `Catch`를 사용하여 상태별로 에러를 처리할 수 있다. 재시도 횟수, 간격, 백오프를 설정 가능하다.

## 시리즈를 마치며

10편에 걸쳐 DVA-C02 시험 범위의 주요 서비스를 정리했다. 시험에서는 서비스 자체의 기능뿐 아니라 **어떤 상황에서 어떤 서비스를 선택해야 하는지**를 묻는 문제가 많다.

- 비동기 처리 → SQS
- 팬아웃 → SNS + SQS
- 실시간 스트리밍 → Kinesis
- 서버리스 API → Lambda + API Gateway
- 사용자 인증 → Cognito
- 비밀 관리 → KMS + Secrets Manager

서비스 간의 관계와 사용 시나리오를 이해하는 것이 합격의 핵심이다.
