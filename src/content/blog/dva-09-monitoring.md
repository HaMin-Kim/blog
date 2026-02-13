---
title: "[DVA-C02 #9] CloudWatch, X-Ray, CloudTrail - 모니터링"
description: "AWS DVA-C02 스터디 9편. AWS의 모니터링, 추적, 감사 서비스를 계층별로 정리합니다."
publishedAt: 2025-05-09
category: "cloud"
tags: ["AWS", "DVA-C02", "CloudWatch", "X-Ray", "CloudTrail"]
---

> AWS DVA-C02 스터디 시리즈 9편입니다. [8편: IaC와 배포](/blog/dva-08-iac-deploy)

## 모니터링 도구 전체 그림

AWS의 모니터링 도구는 역할이 명확하게 나뉜다.

| 도구 | 목적 | 데이터 |
|------|------|--------|
| **CloudWatch** | 인프라/앱 모니터링 | 메트릭, 로그 |
| **X-Ray** | 분산 추적 | 요청의 서비스 간 흐름 |
| **CloudTrail** | API 감사 | 누가 언제 무엇을 호출했는지 |

## CloudWatch Metrics

AWS 리소스의 상태를 수치로 모니터링한다.

### 기본 메트릭

EC2의 경우 CPU, 네트워크, 디스크 I/O가 기본 제공된다. 수집 주기는 5분(무료), 1분(Detailed Monitoring, 유료).

**메모리 사용률은 기본 메트릭에 포함되지 않는다.** CloudWatch Agent를 설치해야 수집할 수 있다.

### 커스텀 메트릭

`PutMetricData` API로 직접 메트릭을 전송할 수 있다.

- Standard Resolution: 60초 간격
- High Resolution: 1초 간격
- Dimension(차원)을 최대 30개까지 붙여 메트릭을 세분화 가능

### 메트릭 보존 기간

| 해상도 | 보존 기간 |
|--------|-----------|
| 1초 (High Resolution) | 3시간 |
| 60초 | 15일 |
| 5분 | 63일 |
| 1시간 | 15개월 |

시간이 지나면 자동으로 낮은 해상도로 집계된다.

## CloudWatch Logs

로그를 수집하고 저장하는 서비스다.

### 구조

- **Log Group**: 로그의 논리적 그룹 (보통 애플리케이션 단위)
- **Log Stream**: Log Group 내의 개별 스트림 (인스턴스/컨테이너 단위)

보존 기간은 1일~10년 또는 무제한으로 설정 가능하다.

### Logs Insights

SQL과 유사한 쿼리 언어로 로그를 분석할 수 있다. 실시간 그래프 생성도 가능해서 간단한 분석에 유용하다.

### Metric Filter

로그에서 특정 패턴을 감지하여 실시간으로 메트릭을 생성한다.

```
예: Nginx 로그에서 5xx 상태 코드를 감지 → "5xxCount" 메트릭 생성 → 알람 설정
```

### Agent 종류

| Agent | 용도 |
|-------|------|
| CloudWatch Logs Agent | 파일/Syslog → Log Group (경량) |
| Unified CloudWatch Agent | 메트릭 + 로그 통합 수집 (권장) |
| Fluent Bit (FireLens) | 컨테이너 로그 라우팅 (ECS/EKS) |

## CloudWatch Alarms

메트릭이 임계값을 넘으면 알림을 보내거나 자동 조치를 수행한다.

### 알람 유형

- **Static Threshold**: 단순 비교 (예: CPU > 80%)
- **Anomaly Detection**: 과거 패턴 기반 이상 탐지
- **Composite Alarm**: 여러 알람을 AND/OR로 조합

### 액션

- SNS로 알림 전송
- ASG 스케일링 트리거
- EC2 인스턴스 복구
- Systems Manager OpsItem 생성

## EventBridge

이벤트 기반 아키텍처의 핵심이다. AWS 서비스, SaaS, 커스텀 앱의 이벤트를 규칙에 따라 타겟으로 라우팅한다.

- **스케줄러**: `cron()`, `at()` 표현식으로 정기 실행
- **이벤트 리플레이**: 최근 24시간의 이벤트를 재전송 가능
- **크로스 리전/계정**: 이벤트를 다른 리전이나 계정으로 전달 가능

## CloudWatch Synthetics

Canary 스크립트(Headless Chrome/Puppeteer)를 실행하여 웹사이트나 API의 가용성을 외부 사용자 관점에서 모니터링한다. 1분~1시간 간격으로 스케줄링할 수 있다.

## X-Ray

마이크로서비스 환경에서 **요청이 서비스 간에 어떻게 흘러가는지** 추적하는 분산 추적 서비스다.

### 핵심 개념

- **Segment**: 하나의 서비스가 처리한 단위
- **Subsegment**: Segment 내의 세부 작업 (DB 쿼리, 외부 API 호출 등)
- **Trace**: 하나의 요청이 거치는 전체 Segment들의 집합

### 적용 방법

| 환경 | 방법 |
|------|------|
| Lambda | 설정만 켜면 자동 적용 |
| EC2/ECS | X-Ray SDK + Daemon 설치 |
| Beanstalk | 콘솔에서 활성화 → Daemon 자동 설치 |
| ECS Fargate | 사이드카 컨테이너로 Daemon 실행 |

### 샘플링

모든 요청을 추적하면 비용이 크므로, 기본적으로 첫 번째 요청 + 이후 5%만 샘플링한다. 샘플링 규칙은 중앙에서 관리하므로 코드 변경 없이 조정 가능하다.

### 주요 API

- `PutTraceSegments`: Segment 데이터 전송
- `PutTelemetryRecords`: 텔레메트리 데이터 전송
- `GetServiceGraph`: 서비스 맵 조회

## CloudTrail

AWS 계정의 **모든 API 호출을 기록**하는 감사 서비스다. 콘솔 클릭, CLI 명령, SDK 호출이 전부 기록된다.

### 용도

- 누가 언제 어떤 리소스를 변경했는지 추적
- 보안 사고 포렌식
- 컴플라이언스 감사

### 주요 기능

- **Management Events**: AWS 리소스 관리 관련 (기본 활성화)
- **Data Events**: S3 객체 접근, Lambda 실행 등 (별도 활성화 필요, 대량 발생)
- **Insight Events**: 비정상적인 API 호출 패턴 탐지
- **EventBridge 연동**: 특정 API 호출 시 실시간 알림/자동 조치

### CloudTrail vs CloudWatch vs X-Ray

| 질문 | 답 |
|------|-----|
| "이 리소스 누가 삭제했어?" | CloudTrail |
| "서버 CPU가 왜 높아?" | CloudWatch |
| "이 API 요청이 왜 느려?" | X-Ray |

다음 글에서는 보안(KMS, Cognito, STS)과 메시징(SQS, SNS, Kinesis) 서비스를 다룬다.
