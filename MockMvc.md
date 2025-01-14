### **테크 블로그 포스트: MockMvc와 @AutoConfigureMockMvc 완벽 가이드**

---

#### **목차**
1. [MockMvc란?](#1-mockmvc란)
2. [@AutoConfigureMockMvc란?](#2-autoconfiguremockmvc란)
3. [MockMvc의 특징 및 동작 방식](#3-mockmvc의-특징-및-동작-방식)
4. [@AutoConfigureMockMvc와 MockMvc 수동 설정의 차이](#4-autoconfiguremockmvc와-mockmvc-수동-설정의-차이)
5. [MockMvc를 활용한 Spring MVC 테스트 코드 작성](#5-mockmvc를-활용한-spring-mvc-테스트-코드-작성)
6. [정리](#6-정리)

---

### **1. MockMvc란?**

`MockMvc`는 Spring MVC 애플리케이션에서 HTTP 요청과 응답을 테스트하기 위한 도구입니다.  
이를 통해 실제 HTTP 요청을 전송하지 않고도 Spring MVC의 구성 요소를 검증할 수 있습니다.

#### **MockMvc의 주요 역할**
- HTTP 요청을 시뮬레이션하여 Spring MVC 요청-응답 흐름을 검증.
- 컨트롤러, 예외 처리기, 필터, 인터셉터 등의 동작 확인.
- 응답의 상태 코드, 헤더, 본문(JSON/XML 등)을 검증.

---

### **2. @AutoConfigureMockMvc란?**

`@AutoConfigureMockMvc`는 Spring Boot에서 MockMvc를 자동으로 설정하는 애노테이션입니다.  
테스트 환경에서 필요한 MockMvc 구성 요소를 자동으로 초기화하여, 별도의 설정 없이 MockMvc를 사용할 수 있게 해줍니다.

#### **@AutoConfigureMockMvc의 역할**
1. **MockMvc Bean 등록**:
   - Spring 컨텍스트에서 MockMvc 객체를 Bean으로 등록합니다.

2. **Spring MVC 구성 자동 초기화**:
   - Spring MVC의 컨트롤러, 예외 처리기, 필터, 인터셉터 등이 포함된 요청-응답 흐름을 테스트 환경에 로드합니다.

3. **테스트 간소화**:
   - MockMvc와 관련된 수동 설정을 생략하고 테스트 작성에 집중할 수 있도록 지원합니다.

---

### **3. MockMvc의 특징 및 동작 방식**

MockMvc의 가장 큰 특징은 **메모리 내에서 요청-응답 흐름을 처리**한다는 점입니다.

#### **3.1. 메모리 내 요청 처리**
- MockMvc는 실제 네트워크 요청을 생성하지 않습니다.
- Spring MVC의 `DispatcherServlet`을 직접 호출하여 요청을 처리하며, 테스트는 메모리 내에서 수행됩니다.

#### **3.2. Spring MVC 요청-응답 흐름 검증**
MockMvc는 다음과 같은 Spring MVC 구성 요소를 검증할 수 있습니다:
1. **컨트롤러**:  
   - HTTP 요청 경로와 메서드(GET, POST 등)에 따른 컨트롤러의 동작 확인.
2. **예외 처리기 (`@ControllerAdvice`)**:  
   - 예외가 발생했을 때 적절한 응답이 반환되는지 검증.
3. **필터와 인터셉터**:  
   - 요청이 컨트롤러에 도달하기 전에 필터와 인터셉터가 올바르게 작동하는지 확인.
4. **AOP 어드바이스**:  
   - 컨트롤러 실행 전후에 동작이 삽입되는지 검증.

#### **3.3. 한계**
- MockMvc는 실제 서블릿 컨테이너(Tomcat, Jetty 등)와의 동작 차이를 확인할 수 없습니다.
- 네트워크 계층과의 통합 테스트가 필요하면 별도의 E2E 테스트 도구를 사용해야 합니다.

---

### **4. @AutoConfigureMockMvc와 MockMvc 수동 설정의 차이**

| 특징                        | @AutoConfigureMockMvc                      | MockMvc 수동 설정 (`MockMvcBuilders`) |
|-----------------------------|-------------------------------------------|---------------------------------------|
| **설정 편의성**              | 자동 설정, 간단한 설정으로 테스트 가능       | 필요한 컨트롤러, 예외 처리기만 선택적 포함 |
| **Spring 컨텍스트 사용 여부**| 전체 Spring 컨텍스트 로드                   | 선택적으로 컨트롤러와 처리기 포함       |
| **Spring Boot 통합**         | Spring Boot와 통합 테스트 가능              | Spring Boot 통합 테스트에는 부적합      |
| **사용 예**                  | MockMvc를 통해 전체 요청-응답 플로우 검증   | 특정 컨트롤러나 동작만 빠르게 검증       |

#### **4.1. @AutoConfigureMockMvc 예제**
```java
@SpringBootTest
@AutoConfigureMockMvc
class AutoConfigMockMvcTest {

    @Autowired
    private MockMvc mockMvc;

    @Test
    void testController() throws Exception {
        mockMvc.perform(get("/api/test"))
               .andExpect(status().isOk())
               .andExpect(content().string("Hello, World!"));
    }
}
```

#### **4.2. MockMvc 수동 설정 예제**
```java
class ManualConfigMockMvcTest {

    private MockMvc mockMvc;

    @BeforeEach
    void setup() {
        this.mockMvc = MockMvcBuilders.standaloneSetup(new TestController())
                                      .setControllerAdvice(new GlobalExceptionHandler())
                                      .build();
    }

    @Test
    void testController() throws Exception {
        mockMvc.perform(get("/api/test"))
               .andExpect(status().isOk())
               .andExpect(content().string("Hello, World!"));
    }
}
```

---

### **5. MockMvc를 활용한 Spring MVC 테스트 코드 작성**

#### **5.1. AOP 어드바이저 테스트**
```java
@Aspect
@Component
public class CustomAspect {

    @Before("execution(* com.example..*(..))")
    public void beforeAdvice(JoinPoint joinPoint) {
        System.out.println("AOP Before Execution");
    }
}
```

**테스트 코드:**
```java
@SpringBootTest
@AutoConfigureMockMvc
class AspectTest {

    @Autowired
    private MockMvc mockMvc;

    @Test
    void testAopExecution() throws Exception {
        mockMvc.perform(get("/api/test"))
               .andExpect(status().isOk())
               .andExpect(content().string("Hello, World!"));
    }
}
```

---

#### **5.2. ControllerAdvice 테스트**
```java
@ControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(Exception.class)
    public ResponseEntity<String> handleException(Exception ex) {
        return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR).body("Error Occurred");
    }
}
```

**테스트 코드:**
```java
@SpringBootTest
@AutoConfigureMockMvc
class ControllerAdviceTest {

    @Autowired
    private MockMvc mockMvc;

    @Test
    void testGlobalExceptionHandler() throws Exception {
        mockMvc.perform(get("/api/error"))
               .andExpect(status().isInternalServerError())
               .andExpect(content().string("Error Occurred"));
    }
}
```

---

### **6. 정리**

1. **MockMvc와 @AutoConfigureMockMvc의 역할**:
   - MockMvc는 Spring MVC 요청-응답 흐름을 메모리 내에서 테스트할 수 있는 강력한 도구입니다.
   - @AutoConfigureMockMvc는 Spring Boot에서 MockMvc를 자동 설정하여 테스트를 간소화합니다.

2. **MockMvc 설정 방법**:
   - Spring Boot 자동 설정을 통해 전체 Spring 컨텍스트를 테스트하거나,
   - 수동 설정으로 특정 컨트롤러와 처리기만 테스트할 수 있습니다.

3. **활용**:
   - AOP 및 ControllerAdvice 테스트에서 MockMvc를 사용하여 예상된 요청-응답 흐름과 동작을 검증할 수 있습니다.

이 글이 MockMvc와 @AutoConfigureMockMvc에 대한 이해를 돕고, 효과적인 Spring MVC 테스트 코드를 작성하는 데 도움이 되길 바랍니다! 😊
