---
title: "의존성 주입, 왜 필요한 건지 처음부터 이해하기"
description: "new로 직접 생성하는 코드에서 출발해서, 생성자 주입, 인터페이스 분리, 그리고 프레임워크가 대신 주입해주는 단계까지. 의존성 주입이 왜 등장했는지를 코드로 따라가봅니다."
publishedAt: 2024-12-15
category: "dev"
tags: ["DI", "OOP", "SOLID", "Spring"]
---

> 이 글은 의존성 주입의 개념과 등장 배경을 다룹니다. Spring에서 DI가 실제로 어떻게 동작하는지(IoC 컨테이너, Bean 스코프, 생명주기 등)는 [Spring IoC 컨테이너와 Bean 생명주기](/blog/spring-ioc-bean-lifecycle) 글에서 이어서 다룹니다.

## 의존성이 뭔데

의존성(Dependency)이라는 단어가 거창하게 느껴질 수 있는데, 사실 별거 아니다.

**A가 B 없이 동작할 수 없으면, A는 B에 의존한다.** 이게 전부다.

```java
public class OrderService {
    private final OrderRepository orderRepository = new OrderRepository();

    public void createOrder(OrderDto dto) {
        orderRepository.save(dto.toEntity());
    }
}
```

`OrderService`는 `OrderRepository` 없이 동작할 수 없다. 그러니까 `OrderService`는 `OrderRepository`에 의존하고 있는 것이다. 여기서 `OrderRepository`가 `OrderService`의 **의존성**이다.

이렇게만 보면 "그래서 뭐가 문제인데?"라는 생각이 들 수 있다. 문제는 의존성 자체가 아니라, **의존성을 다루는 방식**에 있다.

## 직접 생성하는 코드의 문제

위 코드를 다시 보자.

```java
public class OrderService {
    private final OrderRepository orderRepository = new OrderRepository();
}
```

`OrderService`가 `OrderRepository`를 직접 `new`로 생성하고 있다. 이게 왜 문제일까? 간단한 예시에서는 문제가 안 보이니까, 조금 더 현실적인 상황을 만들어보자.

### 테스트를 못 한다

주문 서비스를 테스트하고 싶다. 주문을 생성하면 DB에 저장하고, 결제를 처리하고, 알림을 보내는 로직이다.

```java
public class OrderService {
    private final OrderRepository orderRepository = new OrderRepository();
    private final PaymentService paymentService = new PaymentService();
    private final NotificationService notificationService = new NotificationService();

    public void createOrder(OrderDto dto) {
        Order order = orderRepository.save(dto.toEntity());
        paymentService.process(order);
        notificationService.send(order.getUserEmail(), "주문이 완료되었습니다.");
    }
}
```

이 코드의 단위 테스트를 작성하려면 어떻게 해야 할까?

- `OrderRepository`가 실제 DB에 접근한다. 테스트할 때마다 DB가 필요하다.
- `PaymentService`가 실제 결제를 처리한다. 테스트할 때마다 돈이 빠져나간다.
- `NotificationService`가 실제 이메일을 보낸다. 테스트할 때마다 메일이 간다.

의존성을 직접 생성하면 **가짜(Mock) 객체로 교체할 방법이 없다.** `OrderService` 내부에서 `new`로 만들어버리니까, 외부에서 끼어들 틈이 없는 것이다.

### 구현이 바뀌면 줄줄이 수정해야 한다

`OrderRepository`가 MySQL을 쓰다가 MongoDB로 바꾸게 됐다고 해보자.

```java
// 변경 전
private final OrderRepository orderRepository = new OrderRepository();

// 변경 후 - MySqlOrderRepository를 MongoOrderRepository로 교체
private final MongoOrderRepository orderRepository = new MongoOrderRepository();
```

`OrderRepository`를 사용하는 모든 클래스를 찾아서 수정해야 한다. 사용하는 곳이 10곳이면 10곳 전부. 한 군데라도 빠뜨리면 버그다.

이 두 가지 문제의 본질은 같다. `OrderService`가 **구체적인 구현체에 직접 의존**하고 있기 때문이다.

## 외부에서 넣어주기

해결 방법은 단순하다. `OrderService`가 의존성을 직접 만들지 말고, **밖에서 넣어주면** 된다.

```java
public class OrderService {
    private final OrderRepository orderRepository;
    private final PaymentService paymentService;
    private final NotificationService notificationService;

    // 생성자를 통해 외부에서 주입받는다
    public OrderService(OrderRepository orderRepository,
                         PaymentService paymentService,
                         NotificationService notificationService) {
        this.orderRepository = orderRepository;
        this.paymentService = paymentService;
        this.notificationService = notificationService;
    }

    public void createOrder(OrderDto dto) {
        Order order = orderRepository.save(dto.toEntity());
        paymentService.process(order);
        notificationService.send(order.getUserEmail(), "주문이 완료되었습니다.");
    }
}
```

`new`가 사라졌다. `OrderService`는 이제 의존성이 어떻게 만들어지는지 알 필요가 없다. 그냥 받아서 쓰기만 하면 된다.

이게 **의존성 주입(Dependency Injection)**이다. 의존성을 내부에서 생성하지 않고, 외부에서 주입받는 것.

이제 테스트가 가능해진다.

```java
@Test
void 주문_생성_테스트() {
    // 가짜 객체를 만들어서 넣어줄 수 있다
    OrderRepository mockRepo = mock(OrderRepository.class);
    PaymentService mockPayment = mock(PaymentService.class);
    NotificationService mockNotification = mock(NotificationService.class);

    OrderService orderService = new OrderService(mockRepo, mockPayment, mockNotification);

    // 실제 DB 접근도 없고, 결제도 안 나가고, 메일도 안 간다
    orderService.createOrder(new OrderDto(...));

    verify(mockRepo).save(any());
    verify(mockPayment).process(any());
}
```

외부에서 주입하니까 진짜 객체 대신 Mock을 넣어줄 수 있게 된 것이다.

## 근데 아직 부족하다

의존성을 외부에서 주입받게 바꿨지만, 아직 한 가지 문제가 남아있다.

```java
public class OrderService {
    private final OrderRepository orderRepository;  // 구체 클래스에 의존
    // ...
}
```

`OrderService`가 여전히 `OrderRepository`라는 **구체적인 클래스**에 의존하고 있다. MySQL용 `OrderRepository`를 MongoDB용으로 교체하려면 타입 자체를 바꿔야 한다.

여기서 **인터페이스**가 등장한다.

```java
public interface OrderRepository {
    Order save(Order order);
    Optional<Order> findById(Long id);
}
```

```java
public class MySqlOrderRepository implements OrderRepository {
    @Override
    public Order save(Order order) {
        // MySQL에 저장하는 구현
    }
    // ...
}

public class MongoOrderRepository implements OrderRepository {
    @Override
    public Order save(Order order) {
        // MongoDB에 저장하는 구현
    }
    // ...
}
```

```java
public class OrderService {
    private final OrderRepository orderRepository;  // 인터페이스에 의존

    public OrderService(OrderRepository orderRepository) {
        this.orderRepository = orderRepository;
    }
}
```

이제 `OrderService`는 `OrderRepository`라는 인터페이스에만 의존한다. 실제로 MySQL을 쓰든 MongoDB를 쓰든 `OrderService`는 수정할 필요가 없다. 주입해주는 쪽에서 원하는 구현체를 넣어주면 된다.

```java
// MySQL을 쓰고 싶으면
OrderService service = new OrderService(new MySqlOrderRepository());

// MongoDB로 바꾸고 싶으면
OrderService service = new OrderService(new MongoOrderRepository());
```

`OrderService`의 코드는 한 글자도 안 바꿨다. 이게 인터페이스에 의존하는 것의 힘이다.

## 의존성 역전 원칙 (DIP)

위에서 한 것이 사실 SOLID 원칙 중 하나인 **DIP(Dependency Inversion Principle, 의존성 역전 원칙)**이다.

> 상위 모듈은 하위 모듈에 의존해서는 안 된다. 둘 다 추상화에 의존해야 한다.

말이 좀 어려운데, 코드로 보면 간단하다.

```
[DIP 적용 전]
OrderService → MySqlOrderRepository (상위가 하위의 구체 구현에 의존)

[DIP 적용 후]
OrderService → OrderRepository(인터페이스) ← MySqlOrderRepository
                                           ← MongoOrderRepository
(상위도 하위도 추상화에 의존)
```

의존 방향이 역전됐다. 원래는 `OrderService`가 `MySqlOrderRepository`를 직접 바라보고 있었는데, 이제는 둘 다 `OrderRepository` 인터페이스를 바라보고 있다. 구현체가 바뀌어도 `OrderService`는 영향을 받지 않는다.

## 그런데 누가 주입해주나

의존성을 외부에서 주입받는 건 좋은데, 그러면 결국 **누군가는 객체를 생성하고 연결해줘야** 한다.

```java
// 애플리케이션 시작 시점에 직접 조립해야 한다
OrderRepository orderRepo = new MySqlOrderRepository(dataSource);
PaymentService paymentService = new PaymentService(paymentGateway);
NotificationService notiService = new NotificationService(emailSender);
OrderService orderService = new OrderService(orderRepo, paymentService, notiService);
```

클래스가 몇 개 안 될 때는 괜찮지만, 실제 애플리케이션에서는 수십~수백 개의 객체가 서로 의존하고 있다. 이걸 전부 수동으로 조립하는 건 현실적이지 않다.

그래서 **프레임워크가 대신 해준다.** Spring에서는 이걸 IoC 컨테이너가 담당한다.

```java
@Service
@RequiredArgsConstructor
public class OrderService {
    private final OrderRepository orderRepository;
    private final PaymentService paymentService;
    private final NotificationService notificationService;
    // Spring이 알아서 생성하고 주입해준다
}
```

개발자는 "이 클래스에는 이런 의존성이 필요해요"라고 선언만 하면 되고, 실제 객체의 생성과 주입은 Spring이 처리한다.

## 흐름을 다시 정리하면

의존성 주입은 갑자기 등장한 개념이 아니다. 문제를 해결하는 과정에서 자연스럽게 발전해온 것이다.

```
1단계: 직접 생성 (new)
→ 문제: 테스트 불가, 구현 변경 시 줄줄이 수정

2단계: 외부에서 주입 (생성자 주입)
→ 해결: Mock 교체 가능, 느슨한 결합
→ 남은 문제: 여전히 구체 클래스에 의존

3단계: 인터페이스에 의존 (DIP)
→ 해결: 구현체 변경에 완전히 자유로움
→ 남은 문제: 누가 조립해줄 것인가

4단계: 프레임워크가 대신 주입 (IoC 컨테이너)
→ 해결: 선언만 하면 프레임워크가 알아서 조립
```

각 단계가 이전 단계의 문제를 해결하면서 진화해온 것이다. 의존성 주입을 단순히 "Spring의 기능" 정도로만 알고 있었다면, 이 흐름을 이해하는 게 훨씬 중요하다. 왜 이렇게 하는지를 알아야 적절한 상황에 적절한 설계를 할 수 있으니까.

Spring IoC 컨테이너의 구체적인 동작 방식(Bean 스코프, 생명주기 콜백, 주입 방식 비교 등)은 [Spring IoC 컨테이너와 Bean 생명주기](/blog/spring-ioc-bean-lifecycle) 글에서 이어서 다룬다.
