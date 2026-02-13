---
title: "Spring MVC 요청 처리 흐름, 한 번에 이해하기"
description: "HTTP 요청이 DispatcherServlet에 도착해서 응답이 나가기까지의 전체 흐름을 단계별로 정리합니다."
publishedAt: 2024-12-25
category: "dev"
tags: ["Spring", "MVC", "DispatcherServlet", "Filter"]
---

## 전체 흐름 한눈에 보기

Spring MVC에서 HTTP 요청이 처리되는 전체 흐름은 이렇다.

```
Client → Filter → DispatcherServlet → HandlerMapping → HandlerAdapter
  → Interceptor(preHandle) → Controller → Service → Repository
  → Interceptor(postHandle) → ViewResolver/MessageConverter → Response
```

각 단계가 뭘 하는지 하나씩 살펴보자.

## Filter

요청이 Spring MVC에 도달하기 전, **서블릿 컨테이너(Tomcat) 레벨**에서 먼저 거치는 관문이다.

```java
@Component
public class LoggingFilter implements Filter {
    @Override
    public void doFilter(ServletRequest request, ServletResponse response,
                         FilterChain chain) throws IOException, ServletException {
        HttpServletRequest req = (HttpServletRequest) request;
        log.info("Request: {} {}", req.getMethod(), req.getRequestURI());

        chain.doFilter(request, response);  // 다음 필터 또는 DispatcherServlet으로 전달

        log.info("Response: {}", ((HttpServletResponse) response).getStatus());
    }
}
```

Filter는 Spring 컨텍스트 바깥에서 동작한다. 그래서 Spring Security의 인증/인가 처리, CORS 설정, 요청 로깅 같은 **모든 요청에 공통으로 적용되어야 하는 처리**에 사용된다.

Filter와 Interceptor의 차이를 자주 헷갈려하는데, 핵심은 **실행 시점**이다.

| 구분 | Filter | Interceptor |
|------|--------|-------------|
| 실행 위치 | 서블릿 컨테이너 | Spring MVC |
| 실행 시점 | DispatcherServlet 이전/이후 | Handler 실행 이전/이후 |
| Spring Bean 접근 | 가능 (Spring Boot 환경) | 가능 |
| 주요 용도 | 인증, 인코딩, CORS | 권한 체크, 로깅, API 공통 처리 |

## DispatcherServlet

Spring MVC의 핵심이자 프론트 컨트롤러 패턴의 구현체다. **모든 HTTP 요청의 진입점**이 되어 적절한 핸들러에게 요청을 위임한다.

직접 구현할 일은 거의 없지만, 내부적으로 어떤 순서로 동작하는지 알아두면 디버깅할 때 도움이 된다.

```
1. HandlerMapping에게 요청 URL에 매핑되는 핸들러를 조회한다
2. HandlerAdapter를 통해 핸들러를 실행한다
3. 실행 결과를 적절한 형태로 변환해서 응답한다
```

## HandlerMapping

요청 URL을 보고 어떤 컨트롤러의 어떤 메서드가 처리할지 결정한다.

```java
@RestController
@RequestMapping("/api/users")
public class UserController {

    @GetMapping("/{id}")       // GET /api/users/1 → 이 메서드로 매핑
    public UserDto getUser(@PathVariable Long id) {
        return userService.getUser(id);
    }

    @PostMapping               // POST /api/users → 이 메서드로 매핑
    public UserDto createUser(@RequestBody CreateUserDto dto) {
        return userService.createUser(dto);
    }
}
```

`@RequestMapping`, `@GetMapping` 등의 어노테이션 정보를 기반으로 매핑한다. Spring Boot에서는 `RequestMappingHandlerMapping`이 기본으로 사용된다.

## HandlerAdapter

HandlerMapping이 찾아준 핸들러를 실제로 **실행하는 역할**을 한다. 컨트롤러 메서드의 파라미터를 바인딩하고, 리턴 값을 적절히 처리하는 것도 이 단계에서 일어난다.

```java
@PostMapping
public ResponseEntity<UserDto> createUser(
        @RequestBody CreateUserDto dto,     // 요청 바디 → 객체 변환
        @RequestHeader("X-Token") String token,  // 헤더 값 추출
        @Valid                              // 검증 수행
) {
    return ResponseEntity.ok(userService.createUser(dto));
}
```

`@RequestBody`를 `CreateUserDto`로 변환하고, `@RequestHeader`에서 헤더 값을 꺼내고, `@Valid`로 검증을 수행하는 것이 모두 HandlerAdapter의 `ArgumentResolver`가 처리하는 일이다.

### ArgumentResolver

컨트롤러 메서드의 파라미터에 값을 넣어주는 역할을 한다. 커스텀 어노테이션을 만들어서 자주 쓰는 파라미터 바인딩을 추상화할 수도 있다.

```java
// 커스텀 어노테이션 정의
@Target(ElementType.PARAMETER)
@Retention(RetentionPolicy.RUNTIME)
public @interface LoginUser {}

// ArgumentResolver 구현
public class LoginUserArgumentResolver implements HandlerMethodArgumentResolver {
    @Override
    public boolean supportsParameter(MethodParameter parameter) {
        return parameter.hasParameterAnnotation(LoginUser.class);
    }

    @Override
    public Object resolveArgument(MethodParameter parameter,
                                   ModelAndViewContainer mavContainer,
                                   NativeWebRequest webRequest,
                                   WebDataBinderFactory binderFactory) {
        HttpServletRequest request = webRequest.getNativeRequest(HttpServletRequest.class);
        return request.getAttribute("loginUser");
    }
}

// 컨트롤러에서 사용
@GetMapping("/me")
public UserDto getMyProfile(@LoginUser User user) {
    // user 객체가 자동으로 주입된다
    return UserDto.from(user);
}
```

인증된 사용자 정보를 매번 세션이나 토큰에서 꺼내는 반복 코드를 없앨 수 있어서 실무에서 꽤 자주 쓰인다.

## Interceptor

컨트롤러 실행 전후에 공통 로직을 넣을 수 있다. Filter보다 Spring MVC에 가까운 위치에서 동작한다.

```java
@Component
public class AuthInterceptor implements HandlerInterceptor {

    @Override
    public boolean preHandle(HttpServletRequest request,
                              HttpServletResponse response,
                              Object handler) {
        // 컨트롤러 실행 전
        // false를 리턴하면 컨트롤러를 실행하지 않는다
        String token = request.getHeader("Authorization");
        if (token == null) {
            response.setStatus(401);
            return false;
        }
        return true;
    }

    @Override
    public void postHandle(HttpServletRequest request,
                            HttpServletResponse response,
                            Object handler,
                            ModelAndView modelAndView) {
        // 컨트롤러 실행 후, 뷰 렌더링 전
    }

    @Override
    public void afterCompletion(HttpServletRequest request,
                                 HttpServletResponse response,
                                 Object handler, Exception ex) {
        // 뷰 렌더링까지 완료된 후 (항상 호출됨)
    }
}
```

Interceptor를 등록하려면 `WebMvcConfigurer`를 구현한다.

```java
@Configuration
@RequiredArgsConstructor
public class WebConfig implements WebMvcConfigurer {
    private final AuthInterceptor authInterceptor;

    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(authInterceptor)
                .addPathPatterns("/api/**")
                .excludePathPatterns("/api/auth/**");
    }
}
```

특정 경로에만 적용하거나 제외할 수 있어서 인증 체크, API 버전 관리 등에 유용하다.

## MessageConverter

REST API에서 `@RestController`나 `@ResponseBody`를 사용하면 뷰를 렌더링하는 대신 **객체를 JSON으로 직렬화**해서 응답한다. 이 변환을 담당하는 게 `HttpMessageConverter`다.

```java
@GetMapping("/{id}")
public UserDto getUser(@PathVariable Long id) {
    return userService.getUser(id);
    // UserDto 객체 → Jackson이 JSON으로 변환 → 응답 바디에 담김
}
```

기본적으로 Jackson의 `MappingJackson2HttpMessageConverter`가 등록되어 있어서 별도 설정 없이 JSON 직렬화/역직렬화가 동작한다.

요청 시에도 마찬가지다. `@RequestBody`가 붙은 파라미터는 요청 바디의 JSON을 MessageConverter가 객체로 변환해준다.

## 예외 처리

컨트롤러에서 예외가 발생하면 `@ExceptionHandler`나 `@ControllerAdvice`로 처리할 수 있다.

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(BusinessException.class)
    public ResponseEntity<ErrorResponse> handleBusinessException(BusinessException e) {
        return ResponseEntity
                .status(e.getStatus())
                .body(new ErrorResponse(e.getMessage()));
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ErrorResponse> handleValidationException(
            MethodArgumentNotValidException e) {
        String message = e.getBindingResult().getFieldErrors().stream()
                .map(error -> error.getField() + ": " + error.getDefaultMessage())
                .collect(Collectors.joining(", "));
        return ResponseEntity.badRequest().body(new ErrorResponse(message));
    }
}
```

`@RestControllerAdvice`를 사용하면 모든 컨트롤러에서 발생하는 예외를 한 곳에서 처리할 수 있다. 각 컨트롤러마다 try-catch를 넣는 것보다 훨씬 깔끔하다.

## 전체 흐름 다시 정리

요청이 들어와서 응답이 나가기까지의 흐름을 다시 한번 정리하면 이렇다.

1. **Filter** — 서블릿 레벨에서 요청을 가로채 공통 처리 (인코딩, CORS, Security 등)
2. **DispatcherServlet** — 프론트 컨트롤러로서 전체 흐름을 관장
3. **HandlerMapping** — 요청 URL에 매핑되는 컨트롤러 메서드를 찾음
4. **HandlerAdapter** — 파라미터 바인딩(ArgumentResolver) 후 컨트롤러 메서드 실행
5. **Interceptor** — 컨트롤러 실행 전후에 공통 로직 수행
6. **Controller → Service → Repository** — 실제 비즈니스 로직 처리
7. **MessageConverter** — 반환 객체를 JSON 등으로 변환
8. **ExceptionHandler** — 예외 발생 시 적절한 에러 응답 생성

이 흐름을 한 번 머릿속에 그려두면, "이 처리는 Filter에서 할지 Interceptor에서 할지", "파라미터 바인딩은 어디서 일어나는지", "예외는 어디서 잡아야 하는지" 같은 판단을 내릴 때 기준이 생긴다.
