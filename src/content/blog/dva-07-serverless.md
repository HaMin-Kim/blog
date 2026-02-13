---
title: "[DVA-C02 #7] Lambda, API Gateway, DynamoDB - 서버리스"
description: "AWS DVA-C02 스터디 7편. 서버리스 핵심 3총사인 Lambda, API Gateway, DynamoDB를 정리합니다."
publishedAt: 2025-04-25
category: "cloud"
tags: ["AWS", "DVA-C02", "Lambda", "API Gateway", "DynamoDB"]
---

> AWS DVA-C02 스터디 시리즈 7편입니다. [6편: 컨테이너 서비스](/blog/dva-06-containers)

## Lambda

서버 관리 없이 코드를 실행하는 서버리스 컴퓨팅 서비스다. 요청이 들어올 때만 실행되고, 실행한 만큼만 비용을 지불한다.

### 핵심 특징

- **실행 시간**: 최대 15분
- **메모리**: 128MB ~ 10GB (메모리에 비례하여 CPU도 할당)
- **동시 실행**: 리전당 기본 1,000개 (요청으로 증가 가능)
- **/tmp 스토리지**: 최대 10GB (임시 파일 저장용)
- **환경 변수**: 최대 4KB

### 트리거 소스

Lambda는 다양한 AWS 서비스의 이벤트로 트리거된다.

- **동기 호출**: API Gateway, ALB, CloudFront (Lambda@Edge)
- **비동기 호출**: S3 이벤트, SNS, EventBridge → 실패 시 최대 2회 재시도, DLQ로 전달 가능
- **이벤트 소스 매핑**: SQS, Kinesis, DynamoDB Streams → Lambda가 폴링하여 배치 처리

### Cold Start

Lambda가 처음 실행되거나 오랜만에 실행되면 컨테이너 초기화 시간이 걸린다. 이게 Cold Start다.

완화 방법:
- **Provisioned Concurrency**: 미리 실행 환경을 준비해둔다 (비용 발생)
- **메모리를 늘리면** 초기화도 빨라진다
- 핸들러 밖에서 DB 연결 등 초기화를 수행하면 재사용됨

### Lambda Layer

여러 Lambda 함수에서 공통으로 사용하는 라이브러리를 레이어로 분리할 수 있다. 배포 패키지 크기를 줄이고 라이브러리를 공유하는 데 유용하다.

### VPC 내 Lambda

Lambda를 VPC에 연결하면 프라이빗 서브넷의 RDS, ElastiCache 등에 접근할 수 있다. 단, 인터넷 접근이 필요하면 NAT Gateway를 설정해야 한다.

## API Gateway

REST/WebSocket API를 생성하고 관리하는 서비스다. Lambda의 HTTP 프론트엔드로 가장 많이 사용된다.

### 엔드포인트 유형

- **Regional**: 특정 리전에 배포
- **Edge-Optimized**: CloudFront를 통해 글로벌 배포 (기본값)
- **Private**: VPC 내에서만 접근 가능

### 통합 유형

| 유형 | 동작 |
|------|------|
| Lambda Proxy | 요청을 그대로 Lambda에 전달. 가장 많이 사용 |
| Lambda Custom | 요청/응답 변환(매핑 템플릿) 적용 |
| HTTP Proxy | 외부 HTTP 엔드포인트에 요청 전달 |
| AWS Service | AWS 서비스 API를 직접 호출 |
| Mock | 백엔드 없이 응답 반환 (테스트용) |

### Stage와 배포

API Gateway는 Stage별로 배포한다. `dev`, `staging`, `prod` 같은 환경을 분리할 수 있다. Stage Variable을 사용하면 같은 API 정의로 환경별 다른 Lambda를 호출할 수 있다.

### 캐싱

API 응답을 캐싱하여 백엔드 호출을 줄일 수 있다. TTL(기본 300초), 캐시 크기(0.5GB~237GB)를 설정하며 메서드별로 캐싱을 활성화/비활성화할 수 있다.

### Usage Plan / API Key

API 사용량을 제한하고, API Key 기반으로 클라이언트를 관리할 수 있다. Throttle(초당 요청 수)과 Quota(일/월 요청 수) 제한을 설정한다. 인증 용도가 아닌 **사용량 추적과 제한** 용도다.

### 인증

- **IAM**: AWS 서비스 간 호출 시
- **Cognito User Pool**: 사용자 인증
- **Lambda Authorizer**: 커스텀 인증 로직 (JWT 검증 등)

## DynamoDB

완전 관리형 NoSQL 데이터베이스다. 단일 자릿수 밀리초의 지연 시간을 보장하며, 자동으로 스케일링된다.

### 데이터 모델

- **Table**: 데이터의 집합
- **Item**: 하나의 레코드 (최대 400KB)
- **Attribute**: 항목의 필드

### 키 구조

| 유형 | 구성 | 설명 |
|------|------|------|
| Partition Key (PK) | 단일 키 | 해시 함수로 파티션 결정 |
| Partition Key + Sort Key | 복합 키 | PK로 파티션, SK로 정렬 |

좋은 파티션 키는 **높은 카디널리티**(많은 고유값)를 가져야 한다. 특정 키에 요청이 몰리면 Hot Partition 문제가 발생한다.

### 읽기/쓰기 용량

**Provisioned Mode**: 미리 읽기/쓰기 용량(RCU/WCU)을 설정. 예측 가능한 워크로드에 적합.

**On-Demand Mode**: 사용한 만큼 과금. 예측 불가능하거나 급격한 스파이크가 있는 워크로드에 적합. Provisioned보다 2.5배 비쌈.

### 보조 인덱스

**GSI (Global Secondary Index)**: 원래 테이블과 다른 파티션 키를 사용하는 인덱스. 테이블 생성 후에도 추가 가능. 별도의 RCU/WCU를 소비한다.

**LSI (Local Secondary Index)**: 같은 파티션 키에 다른 소트 키를 사용하는 인덱스. **테이블 생성 시에만** 정의 가능.

### DynamoDB Streams

테이블의 변경사항(INSERT, UPDATE, DELETE)을 실시간 스트림으로 캡처한다. Lambda 트리거와 연결하여 변경 이벤트에 반응하는 로직을 구현할 수 있다.

### DAX (DynamoDB Accelerator)

DynamoDB 전용 인메모리 캐시다. 마이크로초 단위의 읽기 성능을 제공한다. 코드 변경 없이 DAX 클러스터 엔드포인트로 바꾸기만 하면 된다. ElastiCache와의 차이는 DAX가 DynamoDB에 특화되어 있다는 점이다.

### 에러 처리

- **ProvisionedThroughputExceededException**: 용량 초과 시. 지수 백오프(Exponential Backoff)로 재시도.
- Hot Partition 해결: 파티션 키 설계 개선, 쓰기 샤딩(키에 랜덤 suffix 추가)

다음 글에서는 CloudFormation, Elastic Beanstalk 등 IaC와 배포 서비스를 다룬다.
