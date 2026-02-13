---
title: "@Transactional"
description: "Spring @Transactional의 프록시 기반 동작 원리, 전파 속성, self-invocation 문제, rollback 규칙까지 실무에서 자주 마주치는 함정들을 정리합니다."
publishedAt: 2024-12-20
category: "dev"
tags: ["Spring", "Transaction", "AOP", "Proxy"]
---

## @Transactional은 어떻게 동작하는가

`@Transactional`을 메서드에 붙이면 트랜잭션이 알아서 관리된다는 건 대부분 알고 있다. 그런데 "어떻게" 관리되는지를 모르면 실무에서 이상한 버그를 만나게 된다.

Spring의 `@Transactional`은 **프록시 패턴**으로 동작한다. 빈이 등록될 때 원본 객체 대신 프록시 객체가 컨테이너에 등록되고, 외부에서 해당 빈의 메서드를 호출하면 프록시가 가로채서 트랜잭션을 시작하고, 메서드 실행이 끝나면 커밋 또는 롤백을 처리한다.

```
호출자 → [프록시] → 트랜잭션 시작 → [실제 객체의 메서드 실행] → 트랜잭션 커밋/롤백
```

이 구조를 이해하면 아래에서 다룰 문제들이 왜 발생하는지 자연스럽게 이해된다.

## Self-Invocation 문제

Spring을 쓰면서 한 번쯤은 겪게 되는 문제다.

```java
@Service
public class OrderService {

    public void createOrder(OrderDto dto) {
        // 주문 생성 로직
        saveOrder(dto);
    }

    @Transactional
    public void saveOrder(OrderDto dto) {
        orderRepository.save(dto.toEntity());
        paymentRepository.save(dto.toPayment());
    }
}
```

`saveOrder()`에 `@Transactional`을 붙였으니 트랜잭션이 걸릴 것 같지만, `createOrder()`에서 내부 호출하면 **트랜잭션이 적용되지 않는다.**

이유는 간단하다. `createOrder()`를 호출한 건 프록시를 통해 들어왔지만, `saveOrder()`는 `this.saveOrder()`로 호출되기 때문에 프록시를 거치지 않는다. 프록시를 거치지 않으면 트랜잭션 로직이 실행될 수 없다.

해결 방법은 몇 가지가 있다.

```java
// 1. 클래스를 분리한다 (가장 깔끔)
@Service
@RequiredArgsConstructor
public class OrderService {
    private final OrderSaveService orderSaveService;

    public void createOrder(OrderDto dto) {
        orderSaveService.saveOrder(dto);  // 프록시를 통한 호출
    }
}

@Service
public class OrderSaveService {
    @Transactional
    public void saveOrder(OrderDto dto) {
        // ...
    }
}
```

```java
// 2. 자기 자신을 주입받는다 (좀 어색하지만 동작은 한다)
@Service
public class OrderService {
    @Lazy @Autowired
    private OrderService self;

    public void createOrder(OrderDto dto) {
        self.saveOrder(dto);  // 프록시를 통한 호출
    }

    @Transactional
    public void saveOrder(OrderDto dto) {
        // ...
    }
}
```

개인적으로는 클래스 분리를 선호한다. 자기 자신을 주입받는 방식은 코드를 처음 보는 사람이 의도를 파악하기 어렵다.

## 전파 속성 (Propagation)

트랜잭션이 이미 진행 중인 상태에서 또 다른 `@Transactional` 메서드를 호출하면 어떻게 될까? 이걸 결정하는 게 전파 속성이다.

### REQUIRED (기본값)

```java
@Transactional(propagation = Propagation.REQUIRED)  // 기본값이라 생략 가능
public void methodA() {
    methodB();
}

@Transactional
public void methodB() {
    // ...
}
```

기존 트랜잭션이 있으면 참여하고, 없으면 새로 만든다. 대부분의 경우 이걸로 충분하다.

주의할 점은, `methodB()`에서 예외가 발생하면 `methodA()`의 트랜잭션도 함께 롤백된다는 것이다. 같은 트랜잭션을 공유하고 있기 때문이다.

### REQUIRES_NEW

```java
@Transactional
public void methodA() {
    methodB();
    // methodB가 롤백되어도 methodA는 영향 없음
}

@Transactional(propagation = Propagation.REQUIRES_NEW)
public void methodB() {
    // 항상 새로운 트랜잭션에서 실행
}
```

항상 새로운 트랜잭션을 만든다. 기존 트랜잭션은 잠시 보류된다. 로그 저장이나 알림 발송처럼 메인 비즈니스 로직의 성공/실패와 무관하게 독립적으로 처리해야 하는 경우에 사용한다.

### 자주 쓰이는 전파 속성 정리

| 속성 | 동작 |
|------|------|
| `REQUIRED` | 기존 트랜잭션 참여, 없으면 새로 생성 (기본값) |
| `REQUIRES_NEW` | 항상 새로운 트랜잭션 생성 |
| `SUPPORTS` | 기존 트랜잭션 참여, 없으면 트랜잭션 없이 실행 |
| `NOT_SUPPORTED` | 트랜잭션 없이 실행, 기존 트랜잭션은 보류 |
| `MANDATORY` | 기존 트랜잭션 필수, 없으면 예외 발생 |
| `NESTED` | 기존 트랜잭션 내에 세이브포인트를 만들어 부분 롤백 가능 |

실무에서는 `REQUIRED`와 `REQUIRES_NEW`를 가장 많이 쓴다. 나머지는 존재를 알아두고 필요할 때 찾아보면 된다.

## Rollback 규칙

기본적으로 `@Transactional`은 **Unchecked Exception(RuntimeException 하위)** 에서만 롤백한다. Checked Exception은 롤백하지 않는다.

```java
@Transactional
public void example() throws IOException {
    repository.save(entity);
    throw new IOException("체크드 예외");
    // 롤백되지 않는다! 커밋된다.
}
```

이건 꽤 많은 사람이 모르는 부분이다. Checked Exception에서도 롤백하고 싶으면 명시적으로 지정해야 한다.

```java
@Transactional(rollbackFor = Exception.class)
public void example() throws IOException {
    repository.save(entity);
    throw new IOException("체크드 예외");
    // 이제 롤백된다
}
```

반대로 특정 예외에서 롤백하고 싶지 않은 경우도 있다.

```java
@Transactional(noRollbackFor = BusinessException.class)
public void example() {
    // BusinessException이 발생해도 롤백하지 않는다
}
```

## readOnly 옵션

```java
@Transactional(readOnly = true)
public List<User> getUsers() {
    return userRepository.findAll();
}
```

조회 전용 메서드에는 `readOnly = true`를 붙이는 게 좋다. 단순히 "이건 읽기 전용이에요"라고 선언하는 것 이상의 효과가 있다.

- Hibernate가 **더티 체킹을 위한 스냅샷을 저장하지 않는다.** 메모리 절약이 된다.
- DB에 따라 **읽기 전용 트랜잭션 최적화**를 적용하기도 한다. (MySQL의 경우 InnoDB에서 불필요한 락을 줄인다.)
- 의도치 않은 데이터 변경을 방지하는 안전장치 역할도 한다.

## 정리

`@Transactional`은 붙이기만 하면 끝인 것 같지만, 내부 동작을 모르면 트랜잭션이 안 걸리거나, 롤백이 안 되거나, 의도하지 않은 커밋이 발생하는 문제를 겪게 된다.

핵심만 다시 정리하면 이렇다.

- **프록시 기반**이므로 같은 클래스 내부 호출(self-invocation)에서는 트랜잭션이 적용되지 않는다.
- **전파 속성**으로 트랜잭션 간의 관계를 제어할 수 있다. 기본값은 `REQUIRED`.
- **Checked Exception은 기본적으로 롤백하지 않는다.** 필요하면 `rollbackFor`를 명시해야 한다.
- 조회 전용 메서드에는 `readOnly = true`를 붙여서 성능과 안전성을 모두 챙기자.
