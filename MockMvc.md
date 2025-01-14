### **테크 블로그 포스트: MockMvc와 MockMvcBuilders를 사용한 AOP 및 ControllerAdvice 테스트 완벽 가이드**

---

#### **목차**
1. [MockMvc란?](#1-mockmvc란)
2. [MockMvc 사용 배경: AOP와 ControllerAdvice 테스트](#2-mockmvc-사용-배경-aop와-controlleradvice-테스트)
3. [MockMvcBuilders를 사용해야 하는 이유](#3-mockmvcbuilders를-사용해야-하는-이유)
4. [MockMvc의 동작 방식](#4-mockmvc의-동작-방식)
5. [MockMvc 설정 및 테스트 코드 작성](#5-mockmvc-설정-및-테스트-코드-작성)
6. [정리](#6-정리)

---

### **1. MockMvc란?**

`MockMvc`는 Spring MVC 애플리케이션에서 HTTP 요청 및 응답을 테스트하기 위한 도구입니다.  
MockMvc는 실제 HTTP 요청을 생성하지 않고도 Spring MVC의 요청-응답 흐름을 **메모리 내에서 시뮬레이션**하여 다음을 검증할 수 있습니다:

- 컨트롤러 로직의 동작.
- 인터셉터, 필터, AOP와 같은 Spring MVC 구성 요소의 동작.
- HTTP 응답의 상태 코드, 헤더, 본문 검증.

#### **1.1. 한계**
- MockMvc는 실제 서블릿 컨테이너(Tomcat, Jetty 등)와의 동작 차이를 확인할 수 없습니다.
- 네트워크 계층과의 통합 테스트가 필요하면 별도의 E2E 테스트 도구를 사용해야 합니다.

---

### **2. MockMvc 사용 배경: AOP와 ControllerAdvice 테스트**

#### **2.1. AOP의 역할**
- AOP는 특정 조건에 따라 컨트롤러 메서드 호출 전후에 추가적인 동작(로깅, 필터링 등)을 삽입합니다.
- 예를 들어, API 요청의 허용 여부를 확인하는 AOP는 컨트롤러 메서드가 실행되기 전에 조건을 검사하고, 조건이 충족되지 않을 경우 예외를 발생시킵니다.

```java
@Aspect
@Component
public class ApiPermissionAspect {

    @Before("execution(* com.example.controller..*(..))")
    public void checkApiPermission(JoinPoint joinPoint) {
        // API 허용 여부 확인 로직
        System.out.println("AOP Before Controller Execution");
    }
}
```

#### **2.2. ControllerAdvice의 역할**
- ControllerAdvice는 컨트롤러에서 발생한 예외를 처리하고, 적절한 HTTP 응답을 반환합니다.
- 위 AOP에서 발생한 예외를 처리하기 위한 ControllerAdvice는 아래와 같이 작성할 수 있습니다:

```java
@ControllerAdvice
public class ApiExceptionHandler {

    @ExceptionHandler(ApiPermissionException.class)
    public ResponseEntity<String> handleApiPermissionException(ApiPermissionException ex) {
        return ResponseEntity.status(HttpStatus.FORBIDDEN).body("Access Denied: " + ex.getMessage());
    }
}
```

---

### **3. MockMvcBuilders를 사용해야 하는 이유**

Spring Boot에서는 MockMvc를 설정하는 두 가지 방식이 있습니다:
- **자동 설정**: `@AutoConfigureMockMvc`
- **수동 설정**: `MockMvcBuilders`

#### **3.1. @AutoConfigureMockMvc**
`@AutoConfigureMockMvc`는 Spring Boot가 MockMvc를 자동으로 구성하고, Spring MVC 환경을 테스트 컨텍스트에 등록합니다.  
그러나 다음과 같은 이유로 특정 상황에서 피해야 할 수 있습니다:
1. **중복 설정 문제**:
   - `@EnableWebMvc`와 함께 사용할 경우 컨텍스트 초기화 과정에서 Spring MVC와 관련된 설정이 중복되어 예상치 못한 동작을 초래할 수 있습니다.
2. **AOP 및 사용자 정의 설정 간섭**:
   - Spring Boot의 자동 설정이 일부 사용자 정의 AOP 설정을 우회할 가능성이 있습니다.

#### **3.2. MockMvcBuilders**
`MockMvcBuilders`를 사용하면 Spring 컨텍스트 초기화 과정을 명확히 제어할 수 있습니다.
- **webAppContextSetup**:
  - Spring 컨텍스트에 정의된 모든 Bean(AOP 포함)을 포함.
  - 전역 설정(AOP, ControllerAdvice 등)이 적용된 요청-응답 흐름 테스트에 적합.
- **standaloneSetup**:
  - 특정 컨트롤러나 예외 처리기를 독립적으로 테스트.
  - Spring 컨텍스트를 로드하지 않아 초기화 속도가 빠르지만, AOP는 적용되지 않음.

### **3.3. @AutoConfigureMockMvc와 MockMvc 수동 설정의 차이**

| 특징                        | @AutoConfigureMockMvc                      | MockMvc 수동 설정 (`MockMvcBuilders`) |
|-----------------------------|-------------------------------------------|---------------------------------------|
| **설정 편의성**              | 자동 설정, 간단한 설정으로 테스트 가능       | 필요한 컨트롤러, 예외 처리기만 선택적 포함 |
| **Spring 컨텍스트 사용 여부**| 전체 Spring 컨텍스트 로드                   | 선택적으로 컨트롤러와 처리기 포함       |
| **Spring Boot 통합**         | Spring Boot와 통합 테스트 가능              | Spring Boot 통합 테스트에는 부적합      |
| **사용 예**                  | MockMvc를 통해 전체 요청-응답 플로우 검증   | 특정 컨트롤러나 동작만 빠르게 검증       |

---

### **4. MockMvc의 동작 방식**

#### **4.1. 메모리 내 요청-응답 처리**
MockMvc는 Spring MVC의 핵심 구성 요소인 `DispatcherServlet`을 직접 호출하여 요청을 처리합니다.
- 실제 HTTP 네트워크 요청을 사용하지 않아 테스트 속도가 빠릅니다.
- 요청 경로, 메서드, 헤더, 본문을 설정하고 응답 상태 코드와 데이터를 검증할 수 있습니다.

#### **4.2. 요청-응답 플로우**
MockMvc를 사용하면 다음과 같은 Spring MVC 플로우를 검증할 수 있습니다:
1. 요청이 필터나 AOP를 통과하는지 확인.
2. 요청이 올바른 컨트롤러 메서드에 매핑되는지 확인.
3. 예외 발생 시 ControllerAdvice가 작동하는지 확인.

---

### **5. MockMvc 설정 및 테스트 코드 작성**

#### **5.1. TestConfig 설정**
테스트용 Spring 설정을 정의합니다:

```java
@Configuration
@EnableWebMvc
@EnableAspectJAutoProxy
public class TestConfig {

    @Bean
    public ApiPermissionAspect apiPermissionAspect() {
        return new ApiPermissionAspect();
    }

    @Bean
    public ApiExceptionHandler apiExceptionHandler() {
        return new ApiExceptionHandler();
    }

    @RestController
    public static class TestController {
        @GetMapping("/api/test")
        public String testEndpoint() {
            return "API Accessed Successfully!";
        }
    }
}
```

---

#### **5.2. MockMvc 테스트 코드**
`MockMvcBuilders.webAppContextSetup`을 사용하여 Spring 컨텍스트 기반으로 MockMvc를 설정합니다.

```java
@SpringBootTest(classes = TestConfig.class)
class MockMvcAopControllerAdviceTest {

    private MockMvc mockMvc;

    @Autowired
    private WebApplicationContext webApplicationContext;

    @BeforeEach
    void setup() {
        this.mockMvc = MockMvcBuilders.webAppContextSetup(webApplicationContext)
                                      .dispatchOptions(true) // OPTIONS 요청 활성화
                                      .alwaysDo(print()) // 요청/응답 로깅
                                      .build();
    }

    @Test
    void testApiPermissionAspectAndControllerAdvice() throws Exception {
        mockMvc.perform(get("/api/test"))
               .andExpect(status().isOk())
               .andExpect(content().string("API Accessed Successfully!"));
    }
}
```

---

### **6. 정리**

1. **MockMvc의 역할**:
   - Spring MVC 요청-응답 플로우를 테스트할 수 있는 강력한 도구.
   - AOP 및 ControllerAdvice 테스트에 적합.

2. **MockMvc 설정 방법**:
   - Spring Boot의 자동 설정(`@AutoConfigureMockMvc`)은 간편하지만, 사용자 정의 설정이 필요한 경우 `MockMvcBuilders`를 사용.

3. **AOP 및 ControllerAdvice 테스트 시 주의사항**:
   - `MockMvcBuilders.webAppContextSetup`을 사용하여 Spring 컨텍스트의 모든 설정(AOP 포함)을 적용.
   - `@EnableAspectJAutoProxy`와 포인트컷(Pointcut) 설정을 통해 AOP를 활성화.

---
