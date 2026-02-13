---
title: "Spring Security 인증/인가 구조 파헤치기"
description: "Spring Security의 FilterChain 기반 동작 원리, 인증과 인가의 차이, JWT 인증 구현 흐름까지 구조적으로 정리합니다."
publishedAt: 2024-12-28
category: "dev"
tags: ["Spring", "Security", "JWT", "인증"]
---

## 인증과 인가

Spring Security를 이해하려면 먼저 이 두 개념을 확실히 구분해야 한다.

- **인증(Authentication)**: "너 누구야?" — 사용자가 본인이 맞는지 확인하는 과정
- **인가(Authorization)**: "너 이거 할 수 있어?" — 인증된 사용자가 특정 리소스에 접근할 권한이 있는지 확인하는 과정

순서는 항상 인증이 먼저다. 누구인지 모르는데 권한을 확인할 수는 없으니까.

## Spring Security는 Filter 기반이다

Spring Security는 Spring MVC의 컨트롤러에 도달하기 전, **서블릿 필터 체인** 레벨에서 동작한다. 이게 핵심이다.

```
Client → [Servlet Filter Chain] → DispatcherServlet → Controller
              ↑
    Spring Security의 FilterChainProxy가 여기서 동작
```

Spring Security를 의존성에 추가하면 `FilterChainProxy`라는 특수한 필터가 등록되고, 이 안에 보안 관련 필터들이 순서대로 체이닝되어 있다.

기본적으로 등록되는 필터 중 주요한 것들을 순서대로 보면 이렇다.

| 순서 | 필터 | 역할 |
|------|------|------|
| 1 | `SecurityContextPersistenceFilter` | SecurityContext를 로드/저장 |
| 2 | `CorsFilter` | CORS 처리 |
| 3 | `UsernamePasswordAuthenticationFilter` | 폼 로그인 인증 처리 |
| 4 | `BasicAuthenticationFilter` | HTTP Basic 인증 처리 |
| 5 | `ExceptionTranslationFilter` | 인증/인가 예외를 HTTP 응답으로 변환 |
| 6 | `FilterSecurityInterceptor` / `AuthorizationFilter` | 최종 인가 결정 |

전부 외울 필요는 없다. "Security는 필터 체인으로 동작하고, 각 필터가 하나의 보안 관심사를 담당한다"는 구조만 기억하면 된다.

## SecurityFilterChain 설정

Spring Security 6.x(Spring Boot 3.x) 기준으로 보안 설정은 이렇게 작성한다.

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        return http
                .csrf(csrf -> csrf.disable())
                .sessionManagement(session ->
                    session.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
                .authorizeHttpRequests(auth -> auth
                    .requestMatchers("/api/auth/**").permitAll()
                    .requestMatchers("/api/admin/**").hasRole("ADMIN")
                    .anyRequest().authenticated()
                )
                .addFilterBefore(jwtAuthFilter, UsernamePasswordAuthenticationFilter.class)
                .build();
    }
}
```

하나씩 보면,

- `csrf.disable()`: REST API는 세션을 사용하지 않으므로 CSRF 보호가 불필요하다.
- `SessionCreationPolicy.STATELESS`: 세션을 생성하지 않는다. JWT 기반 인증에서는 서버가 상태를 유지하지 않으므로 이 설정이 필요하다.
- `authorizeHttpRequests`: URL별 접근 권한을 설정한다.
- `addFilterBefore`: 커스텀 필터를 특정 위치에 추가한다.

## 인증 처리 흐름

Spring Security의 인증은 `AuthenticationManager` → `AuthenticationProvider` → `UserDetailsService`로 이어지는 구조다.

### 전체 흐름

```
1. 요청에서 인증 정보 추출 (Filter)
2. Authentication 객체 생성
3. AuthenticationManager에게 인증 위임
4. AuthenticationProvider가 실제 인증 수행
5. UserDetailsService에서 사용자 정보 조회
6. 비밀번호 검증 (PasswordEncoder)
7. 인증 성공 시 SecurityContextHolder에 Authentication 저장
```

### UserDetailsService 구현

사용자 정보를 어디서 어떻게 가져올지 정의하는 인터페이스다. 보통 DB에서 조회한다.

```java
@Service
@RequiredArgsConstructor
public class CustomUserDetailsService implements UserDetailsService {
    private final UserRepository userRepository;

    @Override
    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
        User user = userRepository.findByEmail(username)
                .orElseThrow(() -> new UsernameNotFoundException("사용자를 찾을 수 없습니다: " + username));

        return org.springframework.security.core.userdetails.User.builder()
                .username(user.getEmail())
                .password(user.getPassword())
                .roles(user.getRole().name())
                .build();
    }
}
```

Spring Security는 이 인터페이스를 통해 사용자 정보를 가져오고, `PasswordEncoder`로 비밀번호를 검증한다.

## JWT 인증 구현

세션 기반 인증 대신 JWT를 사용하는 경우의 구현 흐름을 정리한다. 실무에서 가장 많이 쓰이는 구조다.

### JWT 유틸리티

```java
@Component
public class JwtProvider {
    @Value("${jwt.secret}")
    private String secretKey;

    @Value("${jwt.expiration}")
    private long expiration;

    public String createToken(String email, String role) {
        Date now = new Date();
        return Jwts.builder()
                .setSubject(email)
                .claim("role", role)
                .setIssuedAt(now)
                .setExpiration(new Date(now.getTime() + expiration))
                .signWith(Keys.hmacShaKeyFor(secretKey.getBytes()), SignatureAlgorithm.HS256)
                .compact();
    }

    public String getEmail(String token) {
        return getClaims(token).getSubject();
    }

    public boolean isValid(String token) {
        try {
            getClaims(token);
            return true;
        } catch (JwtException | IllegalArgumentException e) {
            return false;
        }
    }

    private Claims getClaims(String token) {
        return Jwts.parserBuilder()
                .setSigningKey(Keys.hmacShaKeyFor(secretKey.getBytes()))
                .build()
                .parseClaimsJws(token)
                .getBody();
    }
}
```

### JWT 인증 필터

```java
@Component
@RequiredArgsConstructor
public class JwtAuthenticationFilter extends OncePerRequestFilter {
    private final JwtProvider jwtProvider;
    private final CustomUserDetailsService userDetailsService;

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                     HttpServletResponse response,
                                     FilterChain filterChain)
            throws ServletException, IOException {

        String token = resolveToken(request);

        if (token != null && jwtProvider.isValid(token)) {
            String email = jwtProvider.getEmail(token);
            UserDetails userDetails = userDetailsService.loadUserByUsername(email);

            UsernamePasswordAuthenticationToken authentication =
                    new UsernamePasswordAuthenticationToken(
                            userDetails, null, userDetails.getAuthorities());

            SecurityContextHolder.getContext().setAuthentication(authentication);
        }

        filterChain.doFilter(request, response);
    }

    private String resolveToken(HttpServletRequest request) {
        String bearer = request.getHeader("Authorization");
        if (bearer != null && bearer.startsWith("Bearer ")) {
            return bearer.substring(7);
        }
        return null;
    }
}
```

`OncePerRequestFilter`를 상속받아 요청당 한 번만 실행되도록 한다. 핵심 흐름은 간단하다.

1. `Authorization` 헤더에서 토큰을 꺼낸다.
2. 토큰이 유효하면 사용자 정보를 조회한다.
3. `SecurityContextHolder`에 인증 정보를 저장한다.
4. 이후 Spring Security의 인가 처리에서 이 정보를 활용한다.

### 인증 정보 사용하기

컨트롤러에서 인증된 사용자 정보에 접근하는 방법은 여러 가지다.

```java
// 1. @AuthenticationPrincipal 사용
@GetMapping("/me")
public UserDto getMyProfile(@AuthenticationPrincipal UserDetails userDetails) {
    return userService.getUserByEmail(userDetails.getUsername());
}

// 2. SecurityContextHolder에서 직접 꺼내기
public static String getCurrentUserEmail() {
    Authentication auth = SecurityContextHolder.getContext().getAuthentication();
    return auth.getName();
}
```

## 인가 처리

인증이 완료되면 인가 단계로 넘어간다. "이 사용자가 이 API에 접근할 수 있는가?"를 판단한다.

### URL 기반 인가

`SecurityFilterChain`에서 설정한 URL별 권한이다.

```java
.authorizeHttpRequests(auth -> auth
    .requestMatchers("/api/auth/**").permitAll()        // 누구나 접근 가능
    .requestMatchers("/api/admin/**").hasRole("ADMIN")  // ADMIN만
    .requestMatchers(HttpMethod.DELETE, "/api/**").hasRole("ADMIN")  // DELETE는 ADMIN만
    .anyRequest().authenticated()                        // 나머지는 인증만 되면 OK
)
```

### 메서드 기반 인가

더 세밀한 제어가 필요하면 메서드 레벨에서 권한을 체크할 수 있다.

```java
@Configuration
@EnableMethodSecurity  // 이 설정이 필요하다
public class SecurityConfig { ... }

@Service
public class PostService {

    @PreAuthorize("hasRole('ADMIN')")
    public void deletePost(Long id) {
        // ADMIN만 실행 가능
    }

    @PreAuthorize("#userId == authentication.name")
    public UserDto getUser(String userId) {
        // 본인 정보만 조회 가능
    }
}
```

`@PreAuthorize`는 SpEL(Spring Expression Language)을 사용해서 꽤 유연한 권한 체크가 가능하다.

## 예외 처리

인증/인가에 실패했을 때의 응답을 커스터마이징할 수 있다.

```java
@Component
public class CustomAuthenticationEntryPoint implements AuthenticationEntryPoint {
    @Override
    public void commence(HttpServletRequest request,
                          HttpServletResponse response,
                          AuthenticationException authException) throws IOException {
        response.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
        response.setContentType("application/json;charset=UTF-8");
        response.getWriter().write("{\"message\": \"인증이 필요합니다.\"}");
    }
}

@Component
public class CustomAccessDeniedHandler implements AccessDeniedHandler {
    @Override
    public void handle(HttpServletRequest request,
                        HttpServletResponse response,
                        AccessDeniedException accessDeniedException) throws IOException {
        response.setStatus(HttpServletResponse.SC_FORBIDDEN);
        response.setContentType("application/json;charset=UTF-8");
        response.getWriter().write("{\"message\": \"접근 권한이 없습니다.\"}");
    }
}
```

```java
// SecurityConfig에 등록
http
    .exceptionHandling(exception -> exception
        .authenticationEntryPoint(customAuthenticationEntryPoint)  // 401
        .accessDeniedHandler(customAccessDeniedHandler)             // 403
    )
```

기본 설정으로 두면 HTML 에러 페이지가 반환되기 때문에, REST API에서는 이렇게 JSON 응답으로 커스터마이징하는 것이 일반적이다.

## 정리

Spring Security의 구조를 다시 정리하면 이렇다.

- **Filter 기반**으로 동작한다. 컨트롤러에 도달하기 전 서블릿 레벨에서 보안 처리가 이루어진다.
- **인증**은 "누구인지 확인"이고, **인가**는 "권한이 있는지 확인"이다. 항상 인증이 먼저다.
- JWT 인증은 `OncePerRequestFilter`를 구현해서 토큰 검증 → `SecurityContextHolder`에 인증 정보 저장하는 흐름으로 동작한다.
- 인가는 URL 기반(`authorizeHttpRequests`)과 메서드 기반(`@PreAuthorize`) 두 가지를 상황에 맞게 사용한다.
- REST API에서는 인증/인가 실패 시 JSON 응답을 반환하도록 `EntryPoint`와 `AccessDeniedHandler`를 커스터마이징한다.

Spring Security는 처음 접하면 구조가 복잡하게 느껴지지만, 결국은 필터 체인을 따라 요청이 흘러가는 것이다. 이 흐름만 머릿속에 잡아두면 어떤 보안 요구사항이 와도 "어디에 끼워넣으면 되겠구나"라는 감이 잡힌다.
