---
title: "JPA 영속성 컨텍스트, 제대로 이해하기"
description: "영속성 컨텍스트의 개념부터 더티 체킹, 1차 캐시, 쓰기 지연까지. 단순 암기가 아닌 동작 원리를 중심으로 정리합니다."
publishedAt: 2024-12-13
category: "dev"
tags: ["JPA", "Spring", "영속성 컨텍스트", "Hibernate"]
---

## 영속성 컨텍스트란

영속성 컨텍스트는 엔티티(데이터베이스 테이블과 매핑되는 객체)를 관리하는 메모리 상의 논리적 공간이다.

단순히 "데이터를 임시로 보관하는 곳" 정도로 설명되는 경우가 많은데, 실제로는 그보다 훨씬 많은 일을 한다.

- 조회한 엔티티를 보관하고 (1차 캐시)
- 엔티티의 변경사항을 추적하며 (더티 체킹)
- 변경된 내용을 데이터베이스에 동기화한다 (쓰기 지연)

이 세 가지를 모두 관리하는 가상의 공간이라고 보면 된다. JPA를 쓰면서 "왜 save를 안 했는데 DB에 반영이 되지?" 같은 의문이 들었다면, 그건 영속성 컨텍스트가 뒤에서 일하고 있기 때문이다.

## 엔티티의 생명주기

영속성 컨텍스트를 이해하려면 먼저 엔티티가 어떤 상태를 가지는지 알아야 한다.

### 비영속 (Transient)

```java
User user = new User();
user.setName("김하민");
// 아직 영속성 컨텍스트와 아무 관련이 없는 상태
```

`new`로 생성만 한 상태다. JPA가 이 객체의 존재를 모른다.

### 영속 (Managed)

```java
em.persist(user);
// 또는
User user = userRepository.findById(id);
```

영속성 컨텍스트에 의해 관리되는 상태다. `persist()`로 저장하거나, `findById()` 등으로 조회하면 이 상태가 된다. 이 상태의 엔티티는 변경사항이 자동으로 추적된다.

### 준영속 (Detached)

```java
em.detach(user);  // 특정 엔티티만 분리
em.clear();       // 영속성 컨텍스트 전체 초기화
em.close();       // 영속성 컨텍스트 종료
```

영속 상태였다가 영속성 컨텍스트에서 분리된 상태다. 더 이상 변경사항이 추적되지 않는다. 트랜잭션이 끝난 뒤의 엔티티도 여기에 해당한다.

### 삭제 (Removed)

```java
em.remove(user);
```

삭제가 예약된 상태다. 트랜잭션이 커밋되면 실제 DELETE 쿼리가 나간다.

정리하면 이런 흐름이다.

```
비영속(new) → persist() → 영속(managed) → detach()/clear() → 준영속(detached)
                              ↓
                          remove() → 삭제(removed)
```

이 생명주기를 알아두면 "왜 이 시점에서 쿼리가 안 나가지?", "왜 변경이 반영이 안 되지?" 같은 상황에서 원인을 빠르게 파악할 수 있다.

## 더티 체킹 (Dirty Checking)

영속성 컨텍스트에서 가장 처음 의아하게 느껴지는 부분이 바로 더티 체킹이다.

```java
// 일반적으로 기대하는 업데이트 흐름
User user = userRepository.findById(id);
user.setName("변경된 이름");
userRepository.save(user);  // save를 명시적으로 호출
```

그런데 영속성 컨텍스트가 유지되는 상태에서는 이렇게만 해도 된다.

```java
@Transactional
public void updateUserName(Long id, String newName) {
    User user = userRepository.findById(id).orElseThrow();
    user.setName(newName);
    // save 호출 없이도 트랜잭션 커밋 시점에 UPDATE 쿼리가 나간다
}
```

이게 가능한 이유는 영속성 컨텍스트의 동작 방식 때문이다.

1. 엔티티가 영속 상태로 조회될 때, 그 시점의 상태를 **스냅샷**으로 저장해둔다.
2. 트랜잭션이 커밋되는 시점에 스냅샷과 현재 엔티티의 상태를 비교한다.
3. 변경된 필드가 있으면 해당 필드에 대한 UPDATE 쿼리를 자동으로 생성하고 실행한다.

편리하지만, 이걸 모르고 있으면 의도하지 않은 업데이트가 발생할 수도 있다. 예를 들어 조회한 엔티티의 필드를 실수로 변경하면 트랜잭션이 끝나면서 그 변경이 DB에 반영되어 버린다. "나는 save를 호출한 적이 없는데 왜 데이터가 바뀌었지?" 하는 상황이 이런 케이스다.

## 1차 캐시

영속성 컨텍스트는 내부적으로 1차 캐시를 가지고 있다. 같은 트랜잭션 안에서 동일한 엔티티를 여러 번 조회하면, 첫 번째 조회에서만 실제 쿼리가 나가고 이후에는 캐시에서 가져온다.

```java
@Transactional
public void example(Long id) {
    User user1 = userRepository.findById(id).orElseThrow();  // SELECT 쿼리 실행
    User user2 = userRepository.findById(id).orElseThrow();  // 캐시에서 반환, 쿼리 없음

    System.out.println(user1 == user2);  // true (같은 객체)
}
```

같은 식별자로 조회한 엔티티는 항상 같은 인스턴스를 반환한다. 이걸 **동일성 보장**이라고 한다. `==` 비교가 `true`인 것이다.

다만 1차 캐시는 해당 트랜잭션(또는 EntityManager) 범위 내에서만 유효하다. Redis 같은 애플리케이션 전역 캐시와는 성격이 다르다. 다른 트랜잭션에서 같은 엔티티를 조회하면 당연히 새로 쿼리가 나간다.

## 쓰기 지연 (Write-Behind)

영속성 컨텍스트는 엔티티의 변경사항을 즉시 DB에 반영하지 않는다. 내부적으로 **쓰기 지연 SQL 저장소**에 쿼리를 모아뒀다가, `flush` 시점에 한 번에 DB로 보낸다.

```java
@Transactional
public void createUsers() {
    userRepository.save(new User("유저1"));  // INSERT SQL을 저장소에 보관
    userRepository.save(new User("유저2"));  // INSERT SQL을 저장소에 보관
    userRepository.save(new User("유저3"));  // INSERT SQL을 저장소에 보관

    // 트랜잭션 커밋 시점에 3개의 INSERT가 한 번에 나간다
}
```

이 방식의 장점은 쿼리를 모아서 보내기 때문에 DB와의 통신 횟수를 줄일 수 있다는 것이다. Hibernate의 `hibernate.jdbc.batch_size` 설정과 함께 사용하면 배치 INSERT로 성능을 꽤 끌어올릴 수 있다.

### flush와 commit의 차이

자주 헷갈리는 부분인데, 둘은 다른 개념이다.

- **flush**: 쓰기 지연 SQL 저장소의 쿼리를 DB에 전송한다. 하지만 트랜잭션은 아직 끝나지 않았다.
- **commit**: flush를 먼저 수행한 뒤, 트랜잭션을 확정한다.

flush가 호출되는 시점은 크게 세 가지다.

1. `em.flush()` 직접 호출
2. 트랜잭션 커밋 시점
3. JPQL 쿼리 실행 직전 (JPQL은 DB에 직접 쿼리하므로, 아직 반영 안 된 변경사항이 있으면 먼저 flush 해야 정합성이 맞다)

## 영속성 컨텍스트를 유지하는 방법들

더티 체킹이나 1차 캐시 같은 이점을 누리려면, 엔티티가 영속 상태로 유지되어야 한다. 실무에서 자주 사용하는 방법들을 정리하면 다음과 같다.

### @Transactional

가장 기본적이고 일반적인 방법이다. 트랜잭션이 살아있는 동안 영속성 컨텍스트도 유지된다.

```java
@Transactional
public void updateUser(Long id) {
    User user = userRepository.findById(id).orElseThrow();
    user.setName("변경된 이름");
    // 트랜잭션이 끝나면서 더티 체킹으로 UPDATE 반영
}
```

### OSIV (Open Session In View)

Spring Boot에서 기본적으로 활성화되어 있는 설정이다. HTTP 요청이 시작될 때 영속성 컨텍스트를 열고, 응답이 나갈 때까지 유지한다.

```yaml
spring:
  jpa:
    open-in-view: true  # Spring Boot 기본값
```

컨트롤러나 뷰 레이어에서도 지연 로딩이 가능해지는 편리함이 있지만, DB 커넥션을 오래 들고 있는다는 단점이 있다. 트래픽이 많은 서비스에서는 커넥션 풀이 빠르게 고갈될 수 있어 `false`로 끄는 경우가 많다.

OSIV를 끄면 트랜잭션 밖에서 지연 로딩을 시도할 때 `LazyInitializationException`이 발생한다. 이 경우에는 필요한 연관 데이터를 트랜잭션 안에서 미리 로딩해두어야 한다.

### @EntityGraph

연관 엔티티를 조회 시점에 함께 가져오도록 설정한다. N+1 문제를 해결하는 대표적인 방법 중 하나다.

```java
@EntityGraph(attributePaths = {"posts", "posts.comments"})
Optional<User> findWithPostsById(Long id);
```

fetch join을 어노테이션으로 선언적으로 사용할 수 있어 가독성이 좋다. 다만 복잡한 조건이 필요한 경우에는 JPQL의 `JOIN FETCH`를 직접 사용하는 것이 유연하다.

## 실무에서 주의할 점

### 의도하지 않은 더티 체킹

트랜잭션 안에서 조회한 엔티티를 가공 용도로 사용할 때 주의해야 한다.

```java
@Transactional
public UserDto getUser(Long id) {
    User user = userRepository.findById(id).orElseThrow();
    user.setName(user.getName() + " (탈퇴)");  // 표시용으로 변경한 건데...
    return UserDto.from(user);
    // 트랜잭션이 끝나면서 DB에 "김하민 (탈퇴)"가 저장되어 버린다
}
```

조회 전용 로직에서는 `@Transactional(readOnly = true)`를 사용하면 더티 체킹을 방지할 수 있고, Hibernate가 스냅샷을 저장하지 않으므로 메모리 사용량도 줄어든다.

### 대량 데이터 처리 시 메모리

영속성 컨텍스트는 관리하는 엔티티를 전부 메모리에 올려두기 때문에, 수만 건 이상의 데이터를 한 트랜잭션에서 처리하면 메모리가 부족해질 수 있다.

```java
@Transactional
public void bulkInsert(List<UserDto> users) {
    for (int i = 0; i < users.size(); i++) {
        userRepository.save(users.get(i).toEntity());

        // 일정 단위마다 flush하고 영속성 컨텍스트를 비워준다
        if (i % 500 == 0) {
            entityManager.flush();
            entityManager.clear();
        }
    }
}
```

또는 Spring Data JPA의 `saveAll()`이나, 대량 작업에서는 아예 JDBC batch insert를 직접 사용하는 것도 방법이다.

### N+1 문제

영속성 컨텍스트와 직접적인 관계는 아니지만, 자주 함께 언급되는 문제다. 연관 엔티티가 지연 로딩으로 설정되어 있을 때, 루프를 돌면서 각 엔티티의 연관 데이터에 접근하면 건마다 쿼리가 나간다.

```java
List<User> users = userRepository.findAll();  // 1번 쿼리
for (User user : users) {
    user.getPosts().size();  // User 수만큼 추가 쿼리 (N번)
}
```

이런 경우 앞서 언급한 `@EntityGraph`나 JPQL의 `JOIN FETCH`로 한 번에 가져오도록 해야 한다.

## 정리

영속성 컨텍스트의 핵심 기능을 다시 정리하면 이렇다.

| 기능 | 설명 |
|------|------|
| 1차 캐시 | 같은 트랜잭션 내 동일 엔티티 조회 시 캐시에서 반환, 동일성 보장 |
| 더티 체킹 | 영속 상태 엔티티의 변경사항을 자동 감지하여 UPDATE 반영 |
| 쓰기 지연 | SQL을 모아뒀다가 flush 시점에 일괄 전송 |
| 지연 로딩 | 연관 엔티티를 실제 접근 시점에 조회 (프록시 활용) |

결국 영속성 컨텍스트가 해주는 일은 **개발자가 SQL을 직접 다루는 대신 객체를 다루는 것처럼 코드를 작성할 수 있게 해주는 것**이다. 다만 그 편리함 뒤에서 어떤 동작이 일어나는지 알아야 의도하지 않은 쿼리, 메모리 문제, 데이터 정합성 이슈를 피할 수 있다.
