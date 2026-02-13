---
title: "OpenAPI Generator로 프론트-백엔드 생산성 동시에 높이기"
description: "OpenAPI Generator를 활용한 스키마 퍼스트 개발 방식으로 프론트엔드와 백엔드의 생산성, 안정성을 동시에 확보하는 방법을 정리합니다."
publishedAt: 2024-12-10
category: "dev"
tags: ["OpenAPI", "Spring", "Swagger", "API"]
---

## OpenAPI Generator란?

OpenAPI Generator는 OpenAPI(Swagger) 명세를 기반으로 다양한 언어의 클라이언트 SDK, 서버 스텁, API 문서 등을 자동으로 생성해주는 도구다.

흔히 프론트엔드에서 백엔드 API를 호출하는 함수를 자동 생성하는 용도로 알려져 있지만, 정확히는 **OpenAPI 명세(yaml/json)를 기준으로 코드를 생성하는 방식**이다. 즉 백엔드의 실제 구현이 아닌 명세 문서가 코드 생성의 원본이 된다.

이 구조가 의미하는 바는 다음과 같다.

- **프론트엔드의 생산성을 크게 향상**시킬 수 있다. API 호출 함수, 요청/응답 타입이 자동으로 생성되므로 수작업으로 타입을 정의하거나 API 클라이언트를 작성할 필요가 없다.
- 단, **명세가 곧 코드의 원본**이므로 백엔드에서 스펙에 맞는 정확한 명세를 작성해야 한다. 명세가 부정확하면 생성된 코드도 부정확해진다.

## Schema First vs Code First

명세를 작성할 때 백엔드가 선택할 수 있는 접근 방식은 크게 두 가지다.

### Schema First (스키마 퍼스트)

API 스펙을 먼저 정의하고 명세 문서에 반영한 뒤 구현을 진행하는 방식이다.

- Swagger 문서에 API 엔드포인트와 DTO 구조가 먼저 정의되므로, 프론트엔드가 해당 문서를 보고 **백엔드 구현 완료를 기다리지 않고** 바로 작업에 들어갈 수 있다.
- 명세를 기준으로 mock 데이터를 준비하면 프론트엔드에서도 실제 데이터 기반의 테스트가 가능하다.
- 프론트엔드와 백엔드가 **동일한 명세를 기준으로 개발**하므로 싱크가 어긋나는 문제를 최소화할 수 있다.

### Code First (코드 퍼스트)

구현을 먼저 진행한 뒤 구현된 결과에 맞게 명세에 반영하는 방식이다.

- 프론트엔드는 백엔드 구현이 완료될 때까지 스펙을 확인할 수 없다는 단점이 있다.
- 반면 백엔드 입장에서는 구현 전에 완벽한 스펙을 확정해야 하는 부담이 줄어든다.

### 어떤 방식이 더 좋은가?

상황에 따라 다르지만, 안정성 측면에서는 스키마 퍼스트 방식이 유리하다.

- API 명세가 이미 나와있으므로 프론트/백엔드 간 싱크 불일치를 최소화할 수 있다.
- 명세를 기준으로 mock 서버를 구성하면 프론트엔드가 조기에 데이터 기반 문제에 대응할 수 있다.

그럼에도 현실적으로 코드 퍼스트 방식을 사용하는 팀이 많은데, 이는 구현 과정에서 스펙이 변경될 여지가 크기 때문이다. 특히 초기 설계 단계에서 모든 엣지 케이스를 예측하기 어려운 경우, 구현 중 스펙 변경이 불가피하고 이때마다 명세를 먼저 수정해야 하는 것이 오히려 병목이 될 수 있다.

## 실무에서 시도했던 하이브리드 방식

순수한 스키마 퍼스트는 아니었지만, 완전한 코드 퍼스트도 아닌 방식을 사용했었다.

실제 비즈니스 로직 구현에 들어가기 전 아래 작업을 먼저 수행했다.

1. 반환될 DTO 구조 정의
2. API 엔드포인트 명세 정의
3. Controller 구현체 생성 (인터페이스 수준)
4. DTO에 맞는 mock 데이터 반환

이 상태에서 Swagger 문서화를 마치고, 프론트엔드와 함께 실제 구현 작업에 들어갔다.

이 방식의 장점은 다음과 같았다.

- 실제 구현 전 또는 직후에 동일한 명세 기준으로 프론트엔드와 데이터 구조를 검증할 수 있었다.
- 자잘한 수정에 대해 어느 한쪽만 일방적으로 공수가 들어가는 일 없이 작업 싱크가 잘 맞았다.
- 백엔드에서 잘못된 초기 설계로 스펙이 크게 뒤집히는 것을 방지하기 위해 초기 API/DTO 설계에 많은 공을 들였다.

스키마 퍼스트의 핵심인 "명세를 먼저 공유하여 프론트의 생산성을 높인다"는 점을 어느 정도 달성한 방식이었다.

## OpenAPI Generator를 백엔드에서 활용하기

여기서 한 단계 더 나아가면, 스키마 퍼스트 방식을 활용할 때 **백엔드에서도 OpenAPI Generator를 통해 생산성과 안정성을 동시에 확보**할 수 있다.

OpenAPI Generator의 `spring` 제너레이터를 사용하면 명세로부터 **API 인터페이스와 DTO 클래스**가 자동 생성된다. 컨트롤러는 이 인터페이스를 구현하는 구현체가 되므로, 스펙에 강제되는 안정적인 개발이 가능하다.

### 프로젝트 설정

`build.gradle`에 OpenAPI Generator 플러그인과 관련 의존성을 추가한다.

```groovy
plugins {
    id 'application'
    id 'java'
    id 'org.openapi.generator' version '6.6.0'
    id 'org.springframework.boot' version '3.1.0'
    id 'io.spring.dependency-management' version '1.1.0'
}

dependencies {
    implementation 'org.openapitools:openapi-generator:6.6.0'
    implementation 'io.swagger.core.v3:swagger-annotations:2.2.8'
    implementation 'org.springdoc:springdoc-openapi-starter-webmvc-ui:2.1.0'
}
```

OpenAPI Generator 태스크를 설정한다.

```groovy
openApiGenerate {
    generatorName = "spring"
    inputSpec = "$projectDir/src/main/resources/api.yml"
    outputDir = "$buildDir/generated"
    apiPackage = "com.example.api"
    modelPackage = "com.example.model"
    configOptions = [
        dateLibrary: "java8",
        interfaceOnly: "true",
        useSpringBoot3: "true",
        documentationProvider: "springdoc",
        annotationLibrary: "swagger2"
    ]
}
```

주요 옵션을 살펴보면 다음과 같다.

| 옵션 | 설명 |
|------|------|
| `generatorName` | 사용할 제너레이터. `spring`은 Spring Boot 기반 코드를 생성한다. |
| `inputSpec` | OpenAPI 명세 파일 경로 |
| `interfaceOnly` | `true`로 설정하면 컨트롤러 구현체 없이 인터페이스만 생성한다. |
| `useSpringBoot3` | Spring Boot 3.x 호환 코드를 생성한다. (`jakarta.*` 패키지 사용) |
| `documentationProvider` | Swagger UI 문서화에 사용할 라이브러리 |

선택적으로 Gradle 태스크 의존성을 설정할 수 있다.

```groovy
compileJava.dependsOn('openApiGenerate')
```

이 설정은 Java 컴파일 전에 OpenAPI 코드 생성이 먼저 실행되도록 보장한다. 다만 실무에서는 명세가 변경될 때 수동으로 `openApiGenerate`를 실행하고, 생성된 인터페이스에 맞게 비즈니스 로직도 함께 수정하는 것이 일반적이다. 명세만 변경되고 비즈니스 로직은 수정되지 않은 상태에서 빌드 시 자동 재생성되면 오히려 컴파일 에러의 원인이 될 수 있다.

### OpenAPI 명세 작성

`src/main/resources/api.yml`에 API 스펙을 정의한다.

```yaml
openapi: 3.0.0
info:
  title: Sample API
  version: 1.0.0
paths:
  /users:
    get:
      summary: 사용자 목록 조회
      operationId: getUsers
      responses:
        '200':
          description: 성공
          content:
            application/json:
              schema:
                type: array
                items:
                  $ref: '#/components/schemas/User'

  /pois:
    get:
      summary: 장소 목록 조회
      operationId: getPois
      responses:
        '200':
          description: 성공
          content:
            application/json:
              schema:
                type: array
                items:
                  $ref: '#/components/schemas/Poi'

components:
  schemas:
    User:
      type: object
      properties:
        id:
          type: integer
        name:
          type: string

    Poi:
      type: object
      properties:
        id:
          type: integer
        name:
          type: string
```

### 코드 생성

아래 명령어로 코드를 생성한다.

```bash
./gradlew openApiGenerate
```

`build.gradle`에서 지정한 경로(`$buildDir/generated`)에 API 인터페이스와 DTO가 생성된다.

### 생성된 API 인터페이스

```java
@Generated(value = "org.openapitools.codegen.languages.SpringCodegen")
@Validated
@Tag(name = "pois", description = "the pois API")
public interface PoisApi {

    default Optional<NativeWebRequest> getRequest() {
        return Optional.empty();
    }

    @Operation(
        operationId = "getPois",
        summary = "장소 목록 조회",
        responses = {
            @ApiResponse(responseCode = "200", description = "성공", content = {
                @Content(mediaType = "application/json",
                    array = @ArraySchema(schema = @Schema(implementation = Poi.class)))
            })
        }
    )
    @RequestMapping(
        method = RequestMethod.GET,
        value = "/pois",
        produces = { "application/json" }
    )
    default ResponseEntity<List<Poi>> getPois() {
        getRequest().ifPresent(request -> {
            for (MediaType mediaType : MediaType.parseMediaTypes(request.getHeader("Accept"))) {
                if (mediaType.isCompatibleWith(MediaType.valueOf("application/json"))) {
                    String exampleString = "[ { \"name\" : \"name\", \"id\" : 0 }, { \"name\" : \"name\", \"id\" : 0 } ]";
                    ApiUtil.setExampleResponse(request, "application/json", exampleString);
                    break;
                }
            }
        });
        return new ResponseEntity<>(HttpStatus.NOT_IMPLEMENTED);
    }
}
```

인터페이스에는 `@Operation`, `@ApiResponse` 등 Swagger 어노테이션이 자동으로 포함되며, `default` 메서드로 501 응답을 반환하는 기본 구현이 제공된다. 실제 컨트롤러에서 이 인터페이스를 구현하면 된다.

### 생성된 DTO 클래스

```java
@Generated(value = "org.openapitools.codegen.languages.SpringCodegen")
public class Poi {

    private Integer id;
    private String name;

    public Poi id(Integer id) {
        this.id = id;
        return this;
    }

    @Schema(name = "id", requiredMode = Schema.RequiredMode.NOT_REQUIRED)
    @JsonProperty("id")
    public Integer getId() {
        return id;
    }

    public void setId(Integer id) {
        this.id = id;
    }

    public Poi name(String name) {
        this.name = name;
        return this;
    }

    @Schema(name = "name", requiredMode = Schema.RequiredMode.NOT_REQUIRED)
    @JsonProperty("name")
    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (o == null || getClass() != o.getClass()) return false;
        Poi poi = (Poi) o;
        return Objects.equals(this.id, poi.id) &&
            Objects.equals(this.name, poi.name);
    }

    @Override
    public int hashCode() {
        return Objects.hash(id, name);
    }

    @Override
    public String toString() {
        StringBuilder sb = new StringBuilder();
        sb.append("class Poi {\n");
        sb.append("    id: ").append(toIndentedString(id)).append("\n");
        sb.append("    name: ").append(toIndentedString(name)).append("\n");
        sb.append("}");
        return sb.toString();
    }

    private String toIndentedString(Object o) {
        if (o == null) return "null";
        return o.toString().replace("\n", "\n    ");
    }
}
```

DTO 클래스에는 `@Schema`, `@JsonProperty` 어노테이션이 포함되어 있어 직렬화/역직렬화와 Swagger 문서화가 자동으로 처리된다. Builder 패턴 스타일의 setter(`Poi id(Integer id)`)도 함께 생성되어 테스트 코드 작성 시에도 편리하다.

## 이 방식이 주는 핵심 이점

생성된 API 인터페이스를 컨트롤러에서 `implements`하면, 컨트롤러는 **명세에 정의된 스펙을 강제로 따라야 하는 구현체**가 된다.

```java
@RestController
public class PoisController implements PoisApi {

    @Override
    public ResponseEntity<List<Poi>> getPois() {
        // 실제 비즈니스 로직 구현
    }
}
```

이 구조의 장점은 명확하다.

1. **스펙 준수가 강제된다.** 인터페이스의 반환 타입, 파라미터가 명세와 일치해야 하므로 스펙과 다른 구현이 불가능하다.
2. **Swagger 문서의 정확성이 보장된다.** 이미 OpenAPI 스펙 파일 기반으로 문제없이 인터페이스가 생성된 상태이므로, 해당 스펙이 유효하다는 것이 검증된 셈이다.
3. **프론트엔드의 Generator 오류가 줄어든다.** 백엔드에서 이미 같은 명세로 코드 생성에 성공했으므로, 프론트엔드가 동일 명세로 클라이언트 코드를 생성할 때 오류 가능성이 크게 줄어든다.

## 주의할 점

### 부분 생성을 지원하지 않는다

새로운 API 스펙을 추가하고 `openApiGenerate`를 실행하면, **명세에 정의된 전체 API에 대한 코드가 재생성**된다. 변경된 부분만 선택적으로 생성하는 것은 불가능하다.

인터페이스를 구현하는 컨트롤러 구현체에는 직접적인 영향이 없지만, **DTO 클래스 내부에 커스텀 로직을 추가한 경우 재생성 시 해당 코드가 모두 사라진다.**

이를 방지하기 위한 방법은 다음과 같다.

- DTO는 데이터 구조만 담는 순수한 객체로 유지하고, 변환 로직은 별도의 Mapper나 서비스 레이어에 분리한다.
- 생성된 코드를 직접 수정하지 않고, 필요한 경우 생성된 클래스를 상속하거나 래핑하는 방식으로 확장한다.

## 결론

Swagger를 단순한 API 문서화 도구로만 활용하는 것이 아니라, OpenAPI Generator와 결합하여 프론트엔드의 생산성을 높이려 한다면 **스키마 퍼스트 방식을 도입하는 것이 효과적**이다.

이때 백엔드에서도 OpenAPI Generator를 활용하면 다음의 이점을 얻을 수 있다.

- **안정성**: 명세로부터 생성된 인터페이스를 구현하므로 스펙 준수가 컴파일 레벨에서 강제된다.
- **생산성**: API 인터페이스, DTO 클래스가 자동 생성되어 반복적인 보일러플레이트 코드 작성이 줄어든다.
- **협업 효율**: 프론트엔드와 백엔드가 동일한 명세를 Single Source of Truth로 공유하므로 커뮤니케이션 비용이 줄어든다.
