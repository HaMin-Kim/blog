---
title: "[DVA-C02 #6] ECS, ECR, Fargate - 컨테이너 서비스"
description: "AWS DVA-C02 스터디 6편. Docker 기본 개념부터 ECS, ECR, Fargate, 그리고 EKS까지 컨테이너 오케스트레이션을 정리합니다."
publishedAt: 2025-04-18
category: "cloud"
tags: ["AWS", "DVA-C02", "ECS", "Docker", "Fargate"]
---

> AWS DVA-C02 스터디 시리즈 6편입니다. [5편: VPC와 CloudFront](/blog/dva-05-vpc-cloudfront)

## Docker 기본 개념

Docker는 애플리케이션과 의존성을 하나의 독립된 단위(컨테이너)로 패키징하는 플랫폼이다.

### VM vs Container

- **VM**: CPU, 메모리, OS, 커널을 통째로 가상화. 무겁지만 완전히 격리됨.
- **Container**: 호스트 OS의 커널을 공유. 가볍고 빠르지만 커널 수준의 격리는 아님.

EC2가 VM 방식이라면, ECS/Fargate는 컨테이너 방식이다.

## ECS (Elastic Container Service)

AWS의 컨테이너 오케스트레이션 서비스다. 컨테이너의 배포, 관리, 스케일링을 처리한다.

### 핵심 구성 요소

| 구성 요소 | 역할 |
|-----------|------|
| **Cluster** | 컨테이너가 실행되는 인프라 |
| **Task Definition** | 컨테이너 실행 설정서 (이미지, CPU, 메모리, 포트, 환경변수 등) |
| **Task** | Task Definition 기반으로 실행된 컨테이너 |
| **Service** | 지정된 수의 Task를 유지·관리하는 단위 |

### 시작 유형

**EC2 Launch Type**: 직접 EC2 인스턴스를 관리한다. 인스턴스의 OS, 스케일링, 패치를 직접 해야 하지만 세밀한 제어가 가능하다.

**Fargate Launch Type**: 서버리스. 인프라를 신경 쓸 필요 없이 Task Definition만 정의하면 AWS가 알아서 실행한다. 관리 부담이 없는 대신 EC2보다 비용이 약간 높다.

### Auto Scaling

ECS Service 레벨에서 Task 수를 자동 조절한다.

- **Target Tracking**: 특정 메트릭 목표값 유지 (예: CPU 평균 70%)
- **Step Scaling**: 임계값에 따라 단계적 조절
- **Scheduled Scaling**: 예측 가능한 패턴에 따라 사전 조절

EC2 Launch Type을 쓸 경우, EC2 인스턴스 자체의 ASG도 별도로 관리해야 한다.

### Rolling Update

서비스를 무중단으로 업데이트하는 방식이다.

- **Minimum Healthy Percent**: 업데이트 중 유지해야 하는 최소 Task 비율
- **Maximum Percent**: 업데이트 중 허용되는 최대 Task 비율

예를 들어 min 50%, max 200%이면: 새 Task를 먼저 띄우고 → 헬스 체크 통과 → 이전 Task 종료 순서로 진행된다.

### Task Placement 전략

EC2 Launch Type에서 Task를 어떤 인스턴스에 배치할지 결정한다.

| 전략 | 동작 | 목적 |
|------|------|------|
| binpack | 가장 적은 리소스 여유 인스턴스에 배치 | 비용 최적화 |
| spread | AZ/인스턴스에 균등 분산 | 가용성 |
| random | 랜덤 배치 | 일반적 |

### 아키텍처 패턴

- **웹 앱**: ALB → ECS Service → DB
- **마이크로서비스**: API Gateway → 여러 ECS Service
- **배치 처리**: SQS → ECS Task (큐 기반 워커)
- **CI/CD**: CodeBuild → ECR → ECS 배포

## ECR (Elastic Container Registry)

Docker 이미지를 저장하는 AWS의 프라이빗 레지스트리다.

- IAM 기반 접근 제어
- 이미지 취약점 스캔
- 라이프사이클 정책으로 오래된 이미지 자동 정리
- ECS와 자연스럽게 통합

```bash
# 기본 워크플로우
aws ecr get-login-password | docker login --username AWS --password-stdin <account>.dkr.ecr.<region>.amazonaws.com
docker tag my-app:latest <account>.dkr.ecr.<region>.amazonaws.com/my-app:latest
docker push <account>.dkr.ecr.<region>.amazonaws.com/my-app:latest
```

## EKS (Elastic Kubernetes Service)

AWS의 관리형 Kubernetes 서비스다.

- **ECS vs EKS**: ECS는 AWS에 최적화되어 있어 간단하고, EKS는 표준 Kubernetes라 다른 클라우드로 이식 가능하다. 이미 Kubernetes를 쓰고 있다면 EKS, 새로 시작한다면 ECS가 진입 장벽이 낮다.

## AWS Copilot

컨테이너 애플리케이션 개발을 간소화하는 CLI 도구다. ECS, Fargate 기반 서비스를 몇 가지 명령어로 배포할 수 있다.

```bash
copilot init     # 앱 초기화
copilot deploy   # 배포
copilot svc ls   # 서비스 목록
```

인프라 설정을 자동으로 해주므로 빠른 프로토타이핑에 유용하다.

다음 글에서는 Lambda, API Gateway, DynamoDB 등 서버리스 서비스를 다룬다.
