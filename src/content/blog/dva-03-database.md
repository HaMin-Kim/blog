---
title: "[DVA-C02 #3] RDS, Aurora, ElastiCache - 데이터베이스 서비스"
description: "AWS DVA-C02 스터디 3편. 관리형 RDB 서비스인 RDS/Aurora와 인메모리 캐시 서비스 ElastiCache를 정리합니다."
publishedAt: 2025-04-08
category: "cloud"
tags: ["AWS", "DVA-C02", "RDS", "Aurora", "ElastiCache"]
---

> AWS DVA-C02 스터디 시리즈 3편입니다. [2편: ELB, ASG, Route53](/blog/dva-02-elb-asg-route53)

## RDS (Relational Database Service)

AWS가 관리해주는 관계형 데이터베이스 서비스다. PostgreSQL, MySQL, MariaDB, Oracle, SQL Server, Aurora를 지원한다.

직접 EC2에 DB를 설치하는 것과 비교하면, OS 패칭, 백업, 모니터링, 스케일링 등을 AWS가 알아서 처리해준다는 게 핵심이다.

### Read Replica

읽기 전용 복제본이다. Primary 인스턴스의 읽기 부하를 분산하는 데 사용한다.

- **비동기 복제** (약간의 지연 있음)
- Primary당 최대 **15개**까지 생성 가능
- 다른 AZ, 다른 리전에도 생성 가능
- 독립된 DB로 승격(Promote) 가능
- 같은 리전 내 복제는 네트워크 비용 무료

### Multi-AZ

재해 복구(DR)를 위한 대기 인스턴스 구성이다.

- **동기 복제**. Primary에 쓰기가 발생하면 Standby에도 즉시 반영
- Primary에 장애가 발생하면 **자동 페일오버** (DNS 엔드포인트 유지)
- Standby는 읽기 요청을 처리하지 않음 (페일오버 전까지 대기만)
- Read Replica를 Multi-AZ로 설정할 수도 있음

### RDS Proxy

DB 커넥션 풀링을 관리해주는 프록시 서비스다.

- 커넥션을 풀링하여 DB 부하를 줄여줌
- 페일오버 시 기존 커넥션을 새 인스턴스로 자동 전환 (페일오버 시간 66% 단축)
- IAM 인증 지원
- **Lambda에서 RDS 접근 시 특히 유용**. Lambda는 동시에 수백 개가 실행될 수 있어 커넥션이 폭발할 수 있는데, Proxy가 이걸 관리해줌

## Aurora

AWS가 자체 개발한 고성능 관계형 데이터베이스다. MySQL보다 5배, PostgreSQL보다 3배 빠르다고 한다.

### 핵심 특징

- 스토리지가 **10GB~128TB까지 자동 확장**
- 3개 AZ에 걸쳐 **6개 복제본** 유지
- 쓰기는 하나의 Primary, 읽기는 최대 15개 Replica
- 30초 이내 자동 페일오버
- 비용은 RDS보다 약 20% 높지만 성능 대비 효율적

### Aurora Serverless

사용량에 따라 자동으로 스케일링되는 옵션이다. 트래픽이 예측 불가능하거나, 간헐적으로만 사용되는 워크로드에 적합하다. 사용하지 않을 때는 완전히 꺼져서 비용이 발생하지 않는다.

## ElastiCache

인메모리 캐시 서비스다. Redis와 Memcached를 지원한다.

### Redis vs Memcached

| 기능 | Redis | Memcached |
|------|-------|-----------|
| 데이터 구조 | String, Hash, List, Set 등 | 단순 Key-Value |
| 복제 | Multi-AZ, Read Replica | 없음 |
| 영속성 | AOF, 스냅샷 | 없음 |
| 트랜잭션 | 지원 | 미지원 |
| Pub/Sub | 지원 | 미지원 |
| 멀티스레드 | 단일 스레드 | 멀티스레드 |

대부분의 경우 Redis를 선택한다. Memcached는 단순한 캐싱에 멀티스레드 성능이 필요할 때 고려한다.

### 캐싱 전략

**Lazy Loading (Cache-Aside)**

```
1. 캐시에서 데이터 조회
2. Cache Miss → DB에서 조회 → 캐시에 저장
3. Cache Hit → 캐시에서 바로 반환
```

- 장점: 필요한 데이터만 캐싱, 쓰기 지연 없음
- 단점: Cache Miss 시 3번의 요청 (캐시 조회 → DB 조회 → 캐시 저장), 데이터가 오래될 수 있음

**Write-Through**

```
1. DB에 데이터 저장
2. 동시에 캐시에도 저장
3. 읽기 시 항상 캐시에서 반환
```

- 장점: 캐시 데이터가 항상 최신
- 단점: 모든 쓰기에 캐시 저장 오버헤드, 사용되지 않는 데이터도 캐싱됨

**TTL 전략**: 두 패턴을 조합하면서 TTL(만료 시간)을 설정하는 것이 실무에서 가장 많이 쓰는 방식이다. Lazy Loading으로 캐싱하되, TTL로 오래된 데이터를 자동 만료시킨다.

### 주요 사용 사례

- DB 쿼리 결과 캐싱
- 세션 스토어
- 리더보드 (Redis Sorted Set)
- 실시간 분석
- API 응답 캐싱

다음 글에서는 S3 스토리지를 기초부터 고급 기능까지 다룬다.
