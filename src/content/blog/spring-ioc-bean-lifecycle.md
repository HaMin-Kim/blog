---
title: "Spring IoC 컨테이너와 Bean 생명주기"
description: "의존성 주입이 왜 필요한지부터 생성자 주입을 권장하는 이유, Bean 스코프와 생명주기 콜백까지 한 흐름으로 정리합니다."
publishedAt: 2024-12-22
category: "dev"
tags: ["Spring", "IoC", "DI", "Bean"]
---

> 의존성 주입 자체가 뭔지, 왜 등장했는지부터 알고 싶다면 [의존성 주입, 왜 필요한 건지 처음부터 이해하기](/blog/dependency-injection) 글을 먼저 읽어보는 걸 추천한다.

## 왜 의존성 주입이 필요한가

의존성 주입(DI)의 필요성을 체감하려면 DI 없이 코드를 짜보면 된다.

```java
public class OrderService {
    private final OrderRepository orderRepository = new OrderRepository();
    private final PaymentService paymentService = new PaymentService();

    public void createOrder(OrderDto dto) {
        orderRepository.save(dto.toEntity());
        paymentService.process(dto.toPayment());
    }
}
```

`OrderService`가 직접 `OrderRepository`와 `PaymentService`를 생성하고 있다. 이 코드의 문제는 뭘까?

- `OrderRepository`의 구현이 바뀌면 `OrderService`도 수정해야 한다.
- 테스트할 때 `PaymentService`를 Mock으로 교체할 수가 없다. 실제 결제가 나가버린다.
- 객체의 생성과 사용이 한 곳에 섞여있어서 코드가 경직되어 있다.

의존성 주입은 이 문제를 해결한다. 객체가 자신이 쓸 의존성을 직접 만드는 게 아니라, **외부에서 넣어주는 것**이다.

```java
@Service
@RequiredArgsConstructor
public class OrderService {
    private final OrderRepository orderRepository;
    private final PaymentService paymentService;

    public void createOrder(OrderDto dto) {
        orderRepository.save(dto.toEntity());
        paymentService.process(dto.toPayment());
    }
}
```

`OrderService`는 이제 `OrderRepository`가 어떻게 생성되는지 알 필요가 없다. 테스트할 때 Mock을 넣어줄 수도 있고, 구현체를 바꿔도 `OrderService`를 수정할 필요가 없다.

그리고 이 "외부에서 넣어주는" 역할을 하는 게 바로 **Spring IoC 컨테이너**다.

## IoC 컨테이너가 하는 일

IoC(Inversion of Control)는 제어의 역전이라는 뜻인데, 쉽게 말하면 "객체의 생성과 관리를 개발자가 아니라 프레임워크가 담당한다"는 개념이다.

Spring IoC 컨테이너가 하는 일을 순서대로 보면 이렇다.

1. `@Component`, `@Service`, `@Repository`, `@Controller` 등이 붙은 클래스를 찾는다 (Component Scan)
2. 해당 클래스의 인스턴스를 생성한다
3. 의존 관계를 분석해서 필요한 빈을 주입한다
4. 생성된 빈을 컨테이너에 등록하고 관리한다

개발자는 "이 클래스는 빈이에요"라고 어노테이션으로 선언만 하면 되고, 나머지는 Spring이 알아서 처리한다.

## 의존성 주입 방식 3가지

### 필드 주입

```java
@Service
public class OrderService {
    @Autowired
    private OrderRepository orderRepository;
}
```

코드가 짧아서 편해 보이지만, 테스트할 때 의존성을 외부에서 넣어줄 방법이 없다. (리플렉션을 쓰면 가능하긴 하지만 그건 좋은 방법이 아니다.)

### Setter 주입

```java
@Service
public class OrderService {
    private OrderRepository orderRepository;

    @Autowired
    public void setOrderRepository(OrderRepository orderRepository) {
        this.orderRepository = orderRepository;
    }
}
```

선택적 의존성에 사용할 수 있지만, 빈 생성 시점에 의존성이 주입되지 않으므로 객체가 불완전한 상태로 존재할 수 있다.

### 생성자 주입

```java
@Service
@RequiredArgsConstructor
public class OrderService {
    private final OrderRepository orderRepository;
    private final PaymentService paymentService;
}
```

Spring 공식 문서에서도 권장하는 방식이다. 권장하는 이유가 분명하다.

- `final` 키워드를 사용할 수 있어 **불변성이 보장**된다. 한 번 주입되면 바뀌지 않는다.
- 객체 생성 시점에 모든 의존성이 주입되므로 **NPE를 원천 차단**할 수 있다.
- 생성자의 파라미터가 많아지면 "이 클래스가 너무 많은 책임을 가지고 있구나"라는 신호가 되어 **설계 개선의 힌트**가 된다.
- 테스트 코드에서 직접 생성자를 호출하면서 Mock을 넣어줄 수 있어 **테스트가 쉽다.**

## Bean 스코프

Spring 빈은 기본적으로 **싱글톤**이다. 애플리케이션 전체에서 딱 하나의 인스턴스만 존재한다.

```java
@Service
public class OrderService {
    // 애플리케이션 전체에서 이 인스턴스 하나만 존재
}
```

그런데 상황에 따라 다른 스코프가 필요할 수 있다.

### Singleton (기본값)

컨테이너에 빈이 하나만 생성되고, 모든 요청에서 같은 인스턴스를 공유한다. 대부분의 서비스, 리포지토리가 이 스코프다.

주의할 점은 싱글톤 빈에 **상태(인스턴스 변수)를 두면 안 된다**는 것이다. 여러 스레드가 동시에 접근하므로 동시성 문제가 발생한다.

```java
@Service
public class CounterService {
    private int count = 0;  // 이러면 안 된다. 여러 스레드가 동시에 수정한다.

    public void increment() {
        count++;
    }
}
```

### Prototype

요청할 때마다 새로운 인스턴스를 생성한다.

```java
@Component
@Scope("prototype")
public class PrototypeBean {
    // 주입받을 때마다 새로운 인스턴스
}
```

한 가지 함정이 있다. 싱글톤 빈이 프로토타입 빈을 주입받으면, 싱글톤 빈이 생성될 때 프로토타입 빈도 한 번만 주입되어 이후로는 같은 인스턴스가 계속 사용된다. 매번 새 인스턴스가 필요하다면 `ObjectProvider`를 사용해야 한다.

```java
@Service
@RequiredArgsConstructor
public class SingletonService {
    private final ObjectProvider<PrototypeBean> prototypeBeanProvider;

    public void logic() {
        PrototypeBean bean = prototypeBeanProvider.getObject();  // 매번 새 인스턴스
    }
}
```

### Request

HTTP 요청마다 새로운 인스턴스가 생성되고, 요청이 끝나면 소멸된다.

```java
@Component
@Scope(value = "request", proxyMode = ScopedProxyMode.TARGET_CLASS)
public class RequestScopedBean {
    // 각 HTTP 요청마다 별도 인스턴스
}
```

요청별로 상태를 유지해야 하는 경우(로깅 컨텍스트 등)에 유용하다.

## Bean 생명주기와 콜백

빈이 생성되고 소멸되기까지의 과정을 알아두면 초기화/정리 로직을 적절한 시점에 넣을 수 있다.

```
컨테이너 시작 → 빈 인스턴스 생성 → 의존성 주입 → 초기화 콜백 → 사용 → 소멸 콜백 → 컨테이너 종료
```

### @PostConstruct / @PreDestroy

가장 간단하고 권장되는 방법이다.

```java
@Component
public class CacheManager {
    private Map<String, Object> cache;

    @PostConstruct
    public void init() {
        // 빈 생성 + 의존성 주입 완료 후 호출
        cache = new HashMap<>();
        loadInitialData();
    }

    @PreDestroy
    public void cleanup() {
        // 빈 소멸 전 호출
        cache.clear();
    }
}
```

`@PostConstruct`는 생성자 실행 후 의존성 주입까지 완료된 시점에 호출된다. 생성자에서 초기화 로직을 넣으면 아직 의존성이 주입되지 않은 상태일 수 있으므로, 초기화는 이 콜백에서 하는 것이 안전하다.

### InitializingBean / DisposableBean

```java
@Component
public class CacheManager implements InitializingBean, DisposableBean {
    @Override
    public void afterPropertiesSet() {
        // @PostConstruct와 동일한 시점
    }

    @Override
    public void destroy() {
        // @PreDestroy와 동일한 시점
    }
}
```

Spring 인터페이스에 의존하게 되므로 `@PostConstruct`/@PreDestroy`를 쓰는 게 더 낫다. 이런 방법도 있다는 정도만 알아두면 된다.

## 정리

Spring IoC 컨테이너의 역할을 한 줄로 요약하면, **객체의 생성·관리·주입을 개발자 대신 처리해주는 것**이다.

- **의존성 주입은 생성자 주입**을 사용한다. 불변성, 테스트 용이성, 설계 피드백 세 가지를 모두 챙길 수 있다.
- **빈은 기본적으로 싱글톤**이다. 상태를 가지면 안 된다.
- 프로토타입 빈을 싱글톤에서 사용할 때는 `ObjectProvider`로 감싸야 매번 새 인스턴스를 받을 수 있다.
- **초기화 로직은 `@PostConstruct`**, 정리 로직은 `@PreDestroy`에 넣는다.
