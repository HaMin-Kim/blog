---
title: "[DVA-C02 #5] VPC와 CloudFront - 네트워크와 CDN"
description: "AWS DVA-C02 스터디 5편. VPC의 서브넷, 라우팅, 보안 그룹 구조와 CloudFront CDN의 캐싱 전략을 정리합니다."
publishedAt: 2025-04-15
category: "cloud"
tags: ["AWS", "DVA-C02", "VPC", "CloudFront"]
---

> AWS DVA-C02 스터디 시리즈 5편입니다. [4편: S3 스토리지](/blog/dva-04-s3)

## VPC (Virtual Private Cloud)

AWS 내에서 논리적으로 격리된 가상 네트워크다. 서브넷, 라우팅 테이블, 게이트웨이 등을 직접 설계할 수 있다.

### CIDR 블록

VPC의 IP 주소 범위를 정의한다. 예를 들어 `10.0.0.0/16`이면 10.0.0.0 ~ 10.0.255.255 범위의 65,536개 IP를 사용할 수 있다.

### 서브넷

VPC를 더 작은 네트워크로 나눈 것이다. 각 서브넷은 특정 AZ에 속한다.

- **퍼블릭 서브넷**: 인터넷 게이트웨이(IGW)를 통해 외부와 통신 가능. 웹 서버, ALB 등.
- **프라이빗 서브넷**: 외부 인터넷에서 직접 접근 불가. DB, 내부 서비스 등.

### Internet Gateway (IGW)

VPC와 인터넷을 연결해주는 게이트웨이다. VPC당 하나만 연결 가능하다. 퍼블릭 서브넷의 라우팅 테이블에 `0.0.0.0/0 → IGW` 규칙을 추가해야 인터넷 접근이 가능하다.

### NAT Gateway

프라이빗 서브넷의 리소스가 외부 인터넷으로 **나가는** 통신만 허용한다. 외부에서 들어오는 요청은 차단된다. 소프트웨어 업데이트, 외부 API 호출 등에 사용한다. 퍼블릭 서브넷에 위치해야 한다.

### NACL vs Security Group

| 구분 | NACL | Security Group |
|------|------|----------------|
| 적용 범위 | 서브넷 | 인스턴스/리소스 |
| 상태 | Stateless (인/아웃 각각 규칙 필요) | Stateful (인바운드 허용 시 응답 자동 허용) |
| 규칙 | Allow + Deny | Allow만 |
| 평가 순서 | 번호 순 (낮은 번호 우선) | 전체 동시 평가 |

### VPC Flow Logs

VPC 내 네트워크 트래픽을 캡처한다. 출발지/목적지 IP, 포트, 프로토콜, 허용/거부 여부를 기록한다. CloudWatch Logs나 S3에 저장하여 보안 분석이나 트러블슈팅에 활용한다.

### VPC Peering

서로 다른 VPC를 프라이빗 IP로 연결한다. 같은 계정이든 다른 계정이든 가능하다. 단, **전이적 라우팅(Transitive Routing)은 불가**하다. A↔B, B↔C가 피어링되어도 A↔C는 직접 피어링해야 한다.

### VPC Endpoints

AWS 서비스에 인터넷을 거치지 않고 프라이빗하게 접근할 수 있다.

- **Gateway Endpoint**: S3, DynamoDB 전용. 무료.
- **Interface Endpoint**: ENI 기반. 대부분의 AWS 서비스 지원. 시간당 + 데이터 비용.

## CloudFront

AWS의 CDN(Content Delivery Network) 서비스다. 전 세계 엣지 로케이션에 콘텐츠를 캐싱하여 사용자에게 가장 가까운 위치에서 응답한다.

### 오리진 (Origin)

콘텐츠의 원본 소스다. S3 버킷, ALB, EC2 인스턴스, 외부 HTTP 서버 등을 오리진으로 설정할 수 있다.

### 캐시 키

CloudFront가 캐시를 식별하는 기준이다. 기본적으로 URL이 캐시 키가 된다.

추가로 포함할 수 있는 요소:
- **Query String**: `?size=large` 같은 파라미터
- **Header**: `Accept-Language` 등
- **Cookie**: 세션 기반 분기

캐시 키에 포함되는 요소가 많을수록 캐시 적중률(Hit Rate)이 낮아지므로, 꼭 필요한 것만 포함해야 한다.

### Cache Behavior

URL 경로 패턴별로 다른 캐싱 정책을 적용할 수 있다.

```
/images/*  → S3 오리진, 장기 캐싱
/api/*     → ALB 오리진, 캐싱 없음
/*         → 기본 오리진
```

### Cache Invalidation

엣지 캐시에서 콘텐츠를 강제로 제거한다. 특정 파일(`/index.html`), 패턴(`/images/*`), 전체(`/*`)를 지정할 수 있다. 월 1,000개 경로까지 무료.

대안으로 파일명에 버전을 붙이는 방법(`style-v2.css`)을 쓰면 캐시 무효화 없이도 새 버전을 배포할 수 있다.

### Signed URL / Signed Cookie

프리미엄 콘텐츠, 유료 다운로드 등 특정 사용자에게만 접근을 허용할 때 사용한다.

- **Signed URL**: 개별 파일에 대한 접근 제어
- **Signed Cookie**: 여러 파일에 대한 접근 제어 (URL 변경 없이)

만료 시간, IP 제한 등을 설정할 수 있고, RSA 키 페어로 서명한다.

### Price Class

엣지 로케이션 범위에 따라 비용을 조절할 수 있다.

| Price Class | 범위 | 비용 |
|-------------|------|------|
| All | 전 세계 모든 엣지 | 최고 |
| 200 | 북미, 유럽, 아시아, 중동, 아프리카 | 중간 |
| 100 | 북미, 유럽만 | 최저 |

### Origin Group

Primary 오리진이 실패하면 Secondary 오리진으로 자동 전환하는 장애 복구 메커니즘이다.

### Field-Level Encryption

HTTPS 위에 추가적인 암호화 계층을 제공한다. 특정 필드(신용카드 번호 등)를 엣지에서 공개키로 암호화하고, 백엔드에서만 비공개키로 복호화할 수 있다.

다음 글에서는 Docker와 컨테이너 서비스(ECS, ECR, Fargate)를 다룬다.
