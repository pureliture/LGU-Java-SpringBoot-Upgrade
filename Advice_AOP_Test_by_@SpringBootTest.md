# **Spring Boot 통합 테스트: ControllerAdvice와 AOP**

Spring Boot 애플리케이션에서 **ControllerAdvice**와 **AOP (Aspect-Oriented Programming)** 는 중요한 역할을 합니다. 이 글에서는 **ControllerAdvice와 AOP를 통합 테스트**하는 방법을 다룹니다. 특히 **`@SpringBootTest`**, **`MockMvc`**, **DefaultServletHandler 활성화**, **AOP 및 ControllerAdvice 설정**과 관련된 모든 설정과 테스트 코드를 제시하며, 테스트 중 발생할 수 있는 문제와 해결 방법을 포함합니다.

---

## **ControllerAdvice와 AOP 테스트를 위한 필수 구성 요소**

### 1. **`@SpringBootTest`**
- **필수 이유**:
  - Spring Boot 애플리케이션 컨텍스트를 완전히 로드하여 AOP와 ControllerAdvice를 포함한 모든 Spring 구성 요소를 활성화합니다.
  - 통합 테스트에서 실제 애플리케이션 환경과 유사한 동작을 보장합니다.

---

### 2. **`MockMvc`**
- **필수 이유**:
  - HTTP 요청/응답의 흐름을 시뮬레이션하여 ControllerAdvice와 AOP의 동작을 테스트합니다.
  - 실제 서버를 띄우지 않고도 Spring MVC의 동작을 검증할 수 있습니다.
- **테스트 환경 설정**: `@AutoConfigureMockMvc` 어노테이션을 추가하여 MockMvc를 자동으로 설정합니다.

---

### 3. **DefaultServletHandler 활성화**
- **필수 이유**:
  - DefaultServletHandler를 활성화하면 디스패처 서블릿이 기본 서블릿으로 요청을 위임하여, Spring MVC의 핸들러 체인과 반환값 처리 흐름이 보강됩니다.
  - 통합 테스트 환경에서도 Content Negotiation 및 반환값 직렬화 오류를 방지할 수 있습니다.

---

### 4. **`@EnableWebMvc` 활성화**
- **필수 이유**:
  - Spring MVC 설정을 명시적으로 활성화하여 DefaultServletHandler, Content Negotiation, 정적 리소스 처리 등의 기능을 보장합니다.
  - 테스트 설정에서 `@EnableWebMvc`를 추가하지 않을 경우, Spring MVC의 핵심 기능이 누락될 수 있습니다.

---

### 5. **`@EnableAspectJAutoProxy` 활성화**
- **필수 이유**:
  - Spring AOP를 활성화하여 테스트 중 Aspect의 동작을 검증합니다.
  - AOP 기능을 명시적으로 활성화하지 않으면 `@Aspect`로 정의된 컴포넌트가 동작하지 않을 수 있습니다.

---

### 6. **테스트용 컨트롤러**
- **필수 이유**:
  - 테스트에서 컨트롤러가 없을 경우 AOP와 ControllerAdvice 동작을 검증할 수 없습니다.
  - 테스트 컨트롤러를 별도로 추가하여, 테스트용 엔드포인트를 제공합니다.

---

## **샘플 테스트 코드**

### **1. 설정 코드**

#### TestConfig 설정
`TestConfig`에 DefaultServletHandler 활성화, Spring MVC 기능 활성화(`@EnableWebMvc`), 그리고 AOP 기능 활성화(`@EnableAspectJAutoProxy`)를 추가합니다.

```java
@Configuration
@EnableWebMvc // Spring MVC 기능 활성화
@EnableAspectJAutoProxy // AOP 활성화
public class TestConfig implements WebMvcConfigurer {

    @Override
    public void configureDefaultServletHandling(DefaultServletHandlerConfigurer configurer) {
        configurer.enable(); // DefaultServletHandler 활성화
    }
}
```

---

#### AOP 설정
AOP를 통해 `@GetMapping` 어노테이션이 적용된 메서드 실행 전후에 로그를 출력하는 로직을 추가합니다.

```java
@Aspect
@Component
public class LoggingAspect {

    @Around("@annotation(org.springframework.web.bind.annotation.GetMapping)")
    public Object logExecution(ProceedingJoinPoint joinPoint) throws Throwable {
        System.out.println("Before Execution: " + joinPoint.getSignature().getName());
        Object result = joinPoint.proceed();
        System.out.println("After Execution: " + joinPoint.getSignature().getName());
        return result;
    }
}
```

---

#### ControllerAdvice 설정
예외 발생 시 글로벌 핸들러에서 적절한 HTTP 응답을 반환하도록 설정합니다.

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(RuntimeException.class)
    public ErrorResponseBody handleRuntimeException(RuntimeException ex) {
        return new ErrorResponseBody("server", "03", ex.getMessage(), ex);
    }
}
```

---

#### 테스트용 컨트롤러
테스트를 위해 간단한 컨트롤러를 생성합니다.

```java
@RestController
@RequestMapping("/api")
public class TestController {

    @GetMapping("/success")
    public String successEndpoint() {
        return "Success";
    }

    @GetMapping("/error")
    public String errorEndpoint() {
        throw new RuntimeException("Simulated Error");
    }
}
```

---

### **2. 통합 테스트 코드**

```java
@SpringBootTest(classes = {TestConfig.class, LoggingAspect.class, GlobalExceptionHandler.class, TestController.class})
@AutoConfigureMockMvc // MockMvc 자동 설정
public class IntegrationTest {

    @Autowired
    private MockMvc mockMvc;

    @Test
    public void testSuccessEndpoint() throws Exception {
        mockMvc.perform(get("/api/success"))
               .andExpect(status().isOk())
               .andExpect(content().string("Success"))
               .andDo(print());
        // 로그 출력:
        // Before Execution: successEndpoint
        // After Execution: successEndpoint
    }

    @Test
    public void testErrorEndpointHandledByControllerAdvice() throws Exception {
        mockMvc.perform(get("/api/error")
                        .accept(MediaType.APPLICATION_JSON)) // JSON 응답 기대
               .andExpect(status().isInternalServerError())
               .andExpect(content().contentType(MediaType.APPLICATION_JSON)) // Content-Type 검증
               .andExpect(jsonPath("$.errorCode").value("03"))
               .andExpect(jsonPath("$.errorMsg").value("Simulated Error"))
               .andDo(print());
    }
}
```

---

## **결론**

Spring Boot에서 ControllerAdvice와 AOP를 통합 테스트하려면 다음 요소를 설정해야 합니다:

1. **`@SpringBootTest`**: 애플리케이션 컨텍스트를 완전히 로드.
2. **`@AutoConfigureMockMvc`**: MockMvc 설정을 자동으로 활성화하여 테스트 효율성을 높임.
3. **DefaultServletHandler 활성화**:
   - 반환값 직렬화 문제 방지.
   - Content Negotiation 및 핸들러 체인 개선.
4. **`@EnableWebMvc` 활성화**:
   - Spring MVC의 기능을 명시적으로 활성화.
   - Content Negotiation 및 정적 리소스 처리를 보장.
5. **`@EnableAspectJAutoProxy` 활성화**:
   - AOP의 동작 보장.
6. **테스트용 컨트롤러 추가**:
   - AOP와 ControllerAdvice가 올바르게 동작하는지 확인.
7. **MockMvc를 활용한 테스트 코드 작성**:
   - HTTP 요청/응답 시뮬레이션을 통한 AOP 및 예외 처리 검증.

--- 
