---
title: "[DVA-C02 #8] CloudFormation, Beanstalk, CI/CD - IaC와 배포"
description: "AWS DVA-C02 스터디 8편. Infrastructure as Code 서비스와 CI/CD 파이프라인 구성을 정리합니다."
publishedAt: 2025-05-02
category: "cloud"
tags: ["AWS", "DVA-C02", "CloudFormation", "Beanstalk", "CI/CD"]
---

> AWS DVA-C02 스터디 시리즈 8편입니다. [7편: 서버리스](/blog/dva-07-serverless)

## CloudFormation

YAML/JSON 템플릿으로 AWS 인프라를 정의하고 배포하는 IaC(Infrastructure as Code) 서비스다.

### 왜 쓰는가

- 인프라를 코드로 관리하니 버전 관리, 코드 리뷰가 가능하다
- 동일한 환경을 반복적으로 생성할 수 있다 (dev, staging, prod)
- 리소스 간 의존 관계를 자동으로 처리해준다
- 스택 삭제 시 관련 리소스를 한 번에 정리할 수 있다

### 템플릿 구조

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: 설명

Parameters:     # 외부 입력값
Resources:      # AWS 리소스 정의 (필수)
Mappings:       # 정적 lookup 테이블
Conditions:     # 조건부 리소스 생성
Outputs:        # 스택 결과값 출력
```

### Resources

템플릿의 핵심이다. 생성할 AWS 리소스를 정의한다.

```yaml
Resources:
  MyBucket:                          # 논리적 ID
    Type: AWS::S3::Bucket            # 리소스 타입
    Properties:                      # 설정
      BucketName: my-unique-bucket
```

### Parameters

템플릿을 재사용 가능하게 만든다. 환경별로 다른 값을 주입할 수 있다.

```yaml
Parameters:
  Environment:
    Type: String
    AllowedValues: [dev, staging, prod]
    Default: dev
```

`!Ref` 함수로 파라미터 값을 참조한다: `!Ref Environment`

### Mappings

리전별 AMI ID처럼 정적인 값을 매핑할 때 사용한다.

```yaml
Mappings:
  RegionMap:
    ap-northeast-2:
      AMI: ami-0abcdef1234567890
    us-east-1:
      AMI: ami-0987654321abcdef0
```

`!FindInMap [RegionMap, !Ref "AWS::Region", AMI]`로 값을 가져온다.

### 스택 관리

- **생성**: 템플릿 업로드 → 파라미터 입력 → 리소스 자동 생성
- **업데이트**: 템플릿 수정 → 변경된 리소스만 업데이트
- **삭제**: 스택의 모든 리소스 삭제 (DeletionPolicy로 특정 리소스 보존 가능)

## Elastic Beanstalk

애플리케이션 코드만 올리면 EC2, ASG, ELB, RDS 등을 알아서 구성해주는 PaaS 서비스다.

### 구성 요소

- **Application**: 버전과 환경의 집합
- **Version**: 코드의 각 버전 (S3에 저장)
- **Environment**: 특정 버전이 배포된 인프라 (Web Server / Worker 환경)

### 배포 정책

| 정책 | 다운타임 | 속도 | 롤백 | 비용 |
|------|----------|------|------|------|
| All at once | 있음 | 빠름 | 재배포 | 추가 없음 |
| Rolling | 없음 | 느림 | 재배포 | 추가 없음 |
| Rolling with batch | 없음 | 느림 | 재배포 | 추가 인스턴스 |
| Immutable | 없음 | 느림 | 쉬움 | 추가 인스턴스 |
| Traffic Splitting | 없음 | 느림 | 쉬움 | 추가 인스턴스 |

**Immutable**: 완전히 새로운 인스턴스를 띄우고 트래픽을 전환한다. 롤백이 쉬워서 프로덕션에 적합하다.

**Traffic Splitting**: 새 버전으로 트래픽을 점진적으로 이동한다. 카나리 배포와 비슷하다.

### .ebextensions

`.ebextensions/` 디렉터리에 `.config` 파일(YAML/JSON)을 넣어 환경을 커스터마이징할 수 있다. ElastiCache, RDS 같은 추가 리소스도 정의 가능하다. 내부적으로 CloudFormation을 사용한다.

### 주의할 점

- 로드밸런서 유형은 환경 생성 후 **변경 불가**. 변경하려면 새 환경을 만들어야 한다.
- RDS를 Beanstalk 환경에 포함시키면 환경 삭제 시 **DB도 같이 삭제**된다. 프로덕션에서는 RDS를 별도로 생성하는 것이 안전하다.

## SAM (Serverless Application Model)

서버리스 애플리케이션을 위한 CloudFormation 확장이다. Lambda, API Gateway, DynamoDB를 간결한 문법으로 정의할 수 있다.

```yaml
Transform: AWS::Serverless-2016-10-31

Resources:
  MyFunction:
    Type: AWS::Serverless::Function
    Properties:
      Handler: index.handler
      Runtime: nodejs18.x
      Events:
        Api:
          Type: Api
          Properties:
            Path: /hello
            Method: get
```

CloudFormation으로 직접 쓰면 수십 줄이 필요한 설정을 몇 줄로 표현할 수 있다.

### SAM CLI

```bash
sam init      # 프로젝트 생성
sam build     # 빌드
sam local invoke  # 로컬에서 Lambda 테스트
sam local start-api  # 로컬 API Gateway 실행
sam deploy    # AWS에 배포
```

`sam local`은 Docker를 사용하여 Lambda를 로컬에서 실행한다. 배포 전에 테스트할 수 있어 개발 속도가 빨라진다.

## CDK (Cloud Development Kit)

프로그래밍 언어(TypeScript, Python, Java 등)로 인프라를 정의하는 도구다. 내부적으로 CloudFormation 템플릿으로 변환된다.

### Construct 레벨

| 레벨 | 설명 | 예시 |
|------|------|------|
| L1 | CloudFormation 리소스 1:1 매핑 | CfnBucket |
| L2 | 편의 기능이 추가된 AWS 리소스 | s3.Bucket (기본 설정 포함) |
| L3 | 여러 리소스를 조합한 패턴 | LambdaRestApi (Lambda + API Gateway) |

## CI/CD 파이프라인

### CodeCommit

AWS의 Git 저장소 서비스다. IAM 기반 접근 제어. GitHub의 AWS 버전이라고 보면 된다.

### CodeBuild

소스 코드를 빌드하고 테스트하는 서비스다. `buildspec.yml`에 빌드 과정을 정의한다.

```yaml
version: 0.2
phases:
  install:
    commands:
      - npm install
  build:
    commands:
      - npm run build
      - npm test
artifacts:
  files:
    - '**/*'
  base-directory: dist
```

### CodeDeploy

EC2, ECS, Lambda에 애플리케이션을 배포하는 서비스다.

배포 전략:
- **In-Place**: 기존 인스턴스에 직접 배포 (EC2)
- **Blue/Green**: 새 인스턴스에 배포 후 트래픽 전환
- **Canary**: 트래픽을 단계적으로 이동 (예: 10% → 100%)
- **Linear**: 일정 비율씩 점진적으로 이동 (예: 매 10분마다 10%)

### CodePipeline

위의 서비스들을 연결하여 자동화된 파이프라인을 구성한다.

```
CodeCommit → CodeBuild → CodeDeploy
(Source)      (Build)     (Deploy)
```

각 단계 사이에 Manual Approval을 넣어 사람이 확인 후 진행하도록 할 수도 있다.

다음 글에서는 CloudWatch, X-Ray, CloudTrail 등 모니터링 서비스를 다룬다.
