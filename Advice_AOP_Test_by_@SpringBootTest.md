---

# **Spring Boot 통합 테스트: ControllerAdvice와 AOP**

Spring Boot 애플리케이션에서 **ControllerAdvice**와 **AOP(Aspect-Oriented Programming)** 는 중요한 역할을 합니다. 이 글에서는 두 가지 구성 요소를 통합적으로 테스트하는 방법을 다룹니다. 특히 **`@SpringBootTest`** 를 활용하여 ControllerAdvice와 AOP가 의도한 대로 동작하는지 검증하는 테스트를 작성하는 방법을 설명합니다.

---

## **테스트 클래스에 필요한 요소**

### 1. **`@SpringBootTest`**
- **필수 이유**:
  - Spring Boot 애플리케이션 컨텍스트를 완전히 로드하여, AOP와 ControllerAdvice를 포함한 모든 Spring 구성 요소가 활성화되도록 보장합니다.
  - 테스트 환경이 실제 애플리케이션과 최대한 동일하게 동작합니다.
- **예시**:
  ```java
  @SpringBootTest
  ```

---

### 2. **`@EnableAspectJAutoProxy`**
- **필수 이유**:
  - AOP를 활성화하여 프록시 기반 메서드 호출을 가능하게 합니다.
  - Aspect 클래스에서 정의한 어드바이스가 정상적으로 동작하게 합니다.
- **예시**:
  ```java
  @EnableAspectJAutoProxy
  ```

---

### 3. **`@ControllerAdvice`**
- **필수 이유**:
  - Spring MVC의 글로벌 예외 처리 로직을 테스트할 수 있도록 활성화합니다.
  - 특정 컨트롤러나 모든 컨트롤러에서 발생한 예외를 처리하여 적절한 응답을 반환하는지 검증합니다.
- **예시**:
  ```java
  @ControllerAdvice
  public class GlobalExceptionHandler {
      @ExceptionHandler(RuntimeException.class)
      public ResponseEntity<String> handleRuntimeException(RuntimeException ex) {
          return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR).body("Handled: " + ex.getMessage());
      }
  }
  ```

---

### 4. **AOP (Aspect-Oriented Programming)**
- **필수 이유**:
  - 특정 어노테이션이나 JoinPoint에서 실행되는 공통 로직(예: 로깅, 트랜잭션 관리)이 제대로 적용되는지 확인할 수 있습니다.
- **예시**:
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

### 5. **MockMvc**
- **필수 이유**:
  - HTTP 요청과 응답의 흐름을 시뮬레이션하여, ControllerAdvice와 AOP가 요청 흐름 중 적절히 동작하는지 검증할 수 있습니다.
  - 실제 네트워크 요청 없이 빠른 테스트가 가능합니다.
- **예시**:
  ```java
  MockMvc mockMvc = MockMvcBuilders.webAppContextSetup(context).build();
  ```

---

### 6. **테스트용 컨트롤러**
- **필수 이유**:
  - AOP와 ControllerAdvice가 특정 메서드 호출 시 제대로 동작하는지 검증하려면 테스트 대상 메서드가 필요합니다.
- **예시**:
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

### **통합 테스트 코드**

```java
@ActiveProfiles("test")
@EnableAspectJAutoProxy
@SpringBootTest
@AutoConfigureMockMvc
public class IntegrationTest {

    @Autowired
    private MockMvc mockMvc;

    @Test
    public void testSuccessEndpoint() throws Exception {
        mockMvc.perform(get("/api/success"))
               .andExpect(status().isOk())
               .andExpect(content().string("Success"))
               .andDo(print());

        // 로그 출력 확인:
        // Before Execution: successEndpoint
        // After Execution: successEndpoint
    }

    @Test
    public void testErrorEndpoint() throws Exception {
        mockMvc.perform(get("/api/error"))
               .andExpect(status().isInternalServerError())
               .andExpect(content().string("Handled: Simulated Error"))
               .andDo(print());

        // AOP 및 ControllerAdvice 동작 확인
    }
}
```

---

## **요소별 테스트 결과**

1. **AOP**:
   - `@Around` 어드바이스가 특정 메서드 실행 전후에 호출되어 로그를 출력합니다.
   - 로그 예시:
     ```
     Before Execution: successEndpoint
     After Execution: successEndpoint
     ```

2. **ControllerAdvice**:
   - 예외 발생 시 `GlobalExceptionHandler`에서 적절히 처리하고, HTTP 상태 코드 `500`과 사용자 정의 메시지를 반환합니다.
   - 반환 메시지: `Handled: Simulated Error`

3. **MockMvc**:
   - 요청/응답의 흐름을 검증하며, 컨트롤러와 관련된 모든 동작(AOP, ControllerAdvice)을 통합적으로 테스트합니다.

---

## **결론**

Spring Boot에서 ControllerAdvice와 AOP를 통합 테스트하려면 다음 요소들이 필요합니다:

1. **`@SpringBootTest`**: 전체 컨텍스트 로딩.
2. **`@EnableAspectJAutoProxy`**: AOP 활성화.
3. **`@ControllerAdvice`**: 글로벌 예외 처리 활성화.
4. **MockMvc**: HTTP 요청 시뮬레이션.
5. **테스트 컨트롤러**: AOP와 ControllerAdvice 동작 검증.

위 코드를 통해 AOP와 ControllerAdvice가 의도한 대로 동작하는지 효과적으로 검증할 수 있습니다. Spring Boot 통합 테스트는 실제 환경과 동일한 조건에서 애플리케이션의 안정성을 높이는 데 필수적입니다.

--- 

Spring Boot 애플리케이션의 다른 테스트 전략이 궁금하다면 댓글로 질문 남겨주세요! 😊
