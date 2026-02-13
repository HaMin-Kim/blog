---
title: "[DVA-C02 #2] ELB, ASG, Route53 - 고가용성 아키텍처"
description: "AWS DVA-C02 스터디 2편. 로드밸런서, 오토스케일링, DNS 서비스를 활용한 고가용성 아키텍처 구성을 정리합니다."
publishedAt: 2025-04-04
category: "cloud"
tags: ["AWS", "DVA-C02", "ELB", "ASG", "Route53"]
---

> AWS DVA-C02 스터디 시리즈 2편입니다. [1편: EC2와 인스턴스 스토리지](/blog/dva-01-ec2-storage)

## ELB (Elastic Load Balancing)

트래픽을 여러 인스턴스에 분산시켜주는 서비스다. AWS가 관리해주므로 고가용성, 헬스 체크, SSL 처리를 직접 구현할 필요가 없다.

### ALB (Application Load Balancer)

OSI 7계층(Application)에서 동작한다. HTTP/HTTPS 트래픽을 처리하며, 요청의 내용(URL 경로, 호스트, 헤더 등)을 보고 라우팅할 수 있다.

- URL 기반 라우팅: `/api/*`는 API 서버로, `/static/*`는 정적 서버로
- 호스트 기반 라우팅: `api.example.com`과 `www.example.com`을 다른 타겟으로
- SSL Termination: ALB에서 HTTPS를 복호화하고 백엔드에는 HTTP로 전달
- 헬스 체크로 비정상 인스턴스를 자동 제외

### NLB (Network Load Balancer)

OSI 4계층(Transport)에서 동작한다. TCP/UDP 기반이라 HTTP 헤더를 볼 수 없지만 ALB보다 빠르고, 고정 IP를 할당받을 수 있다.

- 초당 수백만 요청 처리 가능
- 지연 시간이 매우 낮음
- TCP 기반 서비스(게임 서버, IoT 등)에 적합

### GLB (Gateway Load Balancer)

OSI 3계층(Network)에서 동작한다. 트래픽을 보안 어플라이언스(방화벽, IDS 등)로 라우팅할 때 사용한다.

### 주요 기능

**Sticky Session**: 같은 클라이언트의 요청을 항상 같은 인스턴스로 보낸다. 세션 기반 애플리케이션에 필요하지만, 트래픽이 균등하게 분산되지 않는 단점이 있다.

**Cross-Zone Load Balancing**: AZ별 인스턴스 수가 다를 때, AZ 경계를 넘어 전체 인스턴스에 균등 분산한다. ALB는 기본 활성화, NLB는 기본 비활성화.

**SNI (Server Name Indication)**: 하나의 ALB/NLB 리스너에 여러 SSL 인증서를 등록할 수 있다. 요청의 호스트명에 따라 적절한 인증서를 선택한다.

## ASG (Auto Scaling Group)

트래픽에 따라 EC2 인스턴스를 자동으로 늘리고 줄이는 서비스다. ASG 자체는 무료이고, 생성되는 EC2 비용만 지불한다.

### 스케일링 정책

**Dynamic Scaling**: CloudWatch 메트릭 기반으로 실시간 스케일링.

```
예: CPU 사용률이 70%를 넘으면 인스턴스 2개 추가
```

**Simple Scaling**: CloudWatch 알람에 즉시 반응하여 인스턴스 추가/제거.

**Step Scaling**: 메트릭 값의 범위에 따라 단계적으로 스케일링. CPU 70%면 1개, 90%면 3개 추가 같은 방식.

**Scheduled Scaling**: 미리 정해진 시간에 스케일링. 트래픽 패턴이 예측 가능할 때 유용하다.

**Predictive Scaling**: ML이 과거 트래픽 패턴을 분석하여 미리 스케일링을 준비한다.

### 주요 메트릭

- **CPU Utilization**: 가장 많이 사용하는 메트릭
- **RequestCountPerTarget**: 인스턴스당 요청 수 (권장: ~1,000)
- **Network In/Out**: 네트워크 트래픽 기반
- **Memory Utilization**: 기본 제공 아님. CloudWatch Agent 설치 필요

### Cooldown Period

스케일링이 발생한 후 일정 시간 동안 추가 스케일링을 하지 않는 기간이다. 인스턴스가 완전히 올라오기 전에 또 스케일링이 발생하는 것을 방지한다. 기본값은 300초.

## Route53

AWS의 DNS 서비스다. 도메인 이름을 IP 주소로 변환해주는 역할을 한다.

### DNS 기본 개념

도메인은 계층 구조다: Root(.) → TLD(.com) → SLD(example.com) → Subdomain(api.example.com). 각 계층의 네임서버가 다음 계층의 주소를 알려주는 방식으로 최종 IP를 찾아간다.

### TTL (Time-To-Live)

DNS 응답이 캐시되는 시간이다. TTL이 길면 DNS 쿼리 횟수가 줄어 비용이 절감되지만, IP 변경 시 반영이 느리다. 짧으면 변경이 빠르지만 DNS 쿼리가 많아진다.

### 레코드 타입

**CNAME**: 도메인을 다른 도메인에 매핑한다. Zone Apex(example.com 자체)에는 사용할 수 없다.

**ALIAS**: AWS 전용 레코드. CloudFront, S3, ELB 등 AWS 리소스를 가리킬 수 있고, Zone Apex에도 사용 가능하다. DNS 쿼리 비용이 무료.

### 라우팅 정책

| 정책 | 동작 | 사용 사례 |
|------|------|-----------|
| Simple | 단일 IP 반환 (여러 개면 랜덤) | 단순한 매핑 |
| Weighted | 비율에 따라 트래픽 분배 | A/B 테스트, 점진적 배포 |
| Latency-based | 가장 지연이 낮은 리전으로 라우팅 | 글로벌 서비스 |
| Failover | Primary 실패 시 Secondary로 전환 | DR 구성 |
| Geolocation | 사용자 위치 기반 라우팅 | 지역별 콘텐츠 |
| Multi-value | 여러 IP 반환 + 헬스 체크 | 클라이언트 사이드 로드밸런싱 |

다음 글에서는 RDS, Aurora, ElastiCache 등 데이터베이스 서비스를 다룬다.
