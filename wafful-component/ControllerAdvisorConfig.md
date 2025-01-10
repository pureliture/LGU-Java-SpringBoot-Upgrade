### **ControllerAdvisorConfig 클래스 분석**

---

#### **1. 클래스 개요**

`ControllerAdvisorConfig` 클래스는 Spring Boot 애플리케이션에서 컨트롤러 수준의 AOP(Aspect-Oriented Programming)를 설정하고, 공통 동작을 처리하기 위한 설정을 제공합니다. 이 클래스는 요청 본문 처리와 컨트롤러 메서드 호출 전후의 공통 로직을 효율적으로 관리합니다.

---

#### **2. 주요 어노테이션 및 역할**

- **`@AutoConfiguration`**:
  - Spring Boot의 자동 설정 클래스로 등록.
  - Spring Boot 실행 시 이 클래스의 설정이 자동으로 적용됩니다.

- **`@ConditionalOnWebApplication`**:
  - 애플리케이션이 웹 환경에서 실행 중일 때만 이 클래스가 활성화됩니다.

- **`@AutoConfigureOrder(Ordered.LOWEST_PRECEDENCE)`**:
  - 이 설정의 우선순위를 가장 낮게 지정하여 다른 구성보다 나중에 적용됩니다.

- **`@ConditionalOnBean`**:
  - `ControllerAdvisor` 빈이 정의되어 있을 때만 활성화됩니다.

- **`@AutoConfigureAfter(WebConfig.class)`**:
  - `WebConfig` 이후에 이 설정이 적용됩니다.

- **`@EnableAspectJAutoProxy`**:
  - AOP를 활성화하여 프록시 기반의 메서드 호출을 지원합니다.

---

#### **3. 클래스에서 사용된 Spring Framework 컴포넌트**

| Spring 컴포넌트                  | 역할 및 설명                                                                                  |
|----------------------------------|---------------------------------------------------------------------------------------------|
| `FilterRegistrationBean`        | 서블릿 필터를 등록하고, URL 패턴 및 우선순위를 설정합니다.                                     |
| `@ConditionalOnProperty`        | 특정 프로퍼티 조건이 충족될 때만 Bean을 생성합니다.                                             |
| `ObjectProvider`                | Spring 컨텍스트에서 빈을 동적으로 주입하며, 필요 시 빈을 지연 생성합니다.                      |
| `@Bean`                         | Spring 컨텍스트에 빈을 등록하여 다른 컴포넌트에서 의존성을 주입받을 수 있게 합니다.              |

---

#### **4. 주요 메서드 분석**

##### **4.1 HTTP 요청 래퍼 필터 등록**
```java
@Bean
@ConditionalOnProperty(prefix = HttpRequestWrapperPoperties.PREFIX, name = "enabled", havingValue = "false", matchIfMissing = true)
public FilterRegistrationBean<HttpRequestWrapperFilter> httpRequestWapperFilterRegistrationBeanForHttpBizInterceptor() {
    HttpRequestWrapperFilter filter = new HttpRequestWrapperFilter();
    FilterRegistrationBean<HttpRequestWrapperFilter> registrationBean = new FilterRegistrationBean<>(filter);
    registrationBean.addUrlPatterns("/*");
    registrationBean.setOrder(filter.getOrder());
    return registrationBean;
}
```

- **설명**:
  - `HttpRequestWrapperFilter`를 등록하여 요청 본문(`RequestBody`)을 여러 번 읽을 수 있도록 지원합니다.
  - `FilterRegistrationBean`을 사용하여 필터를 등록하며, URL 패턴과 우선순위를 설정합니다.
- **조건**:
  - 프로퍼티 `wafful.http-request-wrapper-filter.enabled`가 `false`일 때 활성화됩니다.
  - 기본값은 `false`입니다(`matchIfMissing = true`).

- **Spring 컴포넌트**:
  - `FilterRegistrationBean`: 서블릿 필터 등록을 지원.
  - `@ConditionalOnProperty`: 프로퍼티 값에 따라 필터 활성화 여부를 제어.

---

##### **4.2 컨트롤러 AOP 등록**
```java
@Bean
public ControllerAdvisorDecorateAspect controllerAdvisorDecorateAspect(ObjectProvider<ControllerAdvisor> dbSelectHttpAssistantProvider) {
    return new ControllerAdvisorDecorateAspect(dbSelectHttpAssistantProvider);
}
```

- **설명**:
  - `ControllerAdvisorDecorateAspect`를 등록하여 컨트롤러 메서드 호출 시 AOP를 적용합니다.
  - `ObjectProvider`를 사용하여 필요 시 `ControllerAdvisor` 빈을 지연 로드합니다.
  - `@GetMapping`, `@PostMapping` 등 컨트롤러 메서드 호출 전후에 특정 동작(예: 로깅, 데이터 검증)을 수행합니다.
- **Spring 컴포넌트**:
  - `ObjectProvider`: 지연 로드와 의존성 주입을 지원.
  - `@Bean`: Spring 컨텍스트에 AOP 관련 빈 등록.

---

#### **5. 강점**

1. **유연한 AOP 설정**:
   - `ObjectProvider`를 사용하여 의존성을 지연 로드하고, 필요에 따라 `ControllerAdvisor`를 제공.
   - AOP를 통해 공통 동작을 컨트롤러 메서드 호출 전후에 쉽게 적용.

2. **요청 본문 처리 개선**:
   - `HttpRequestWrapperFilter`를 통해 요청 본문을 여러 번 읽을 수 있도록 지원하여 데이터 처리의 유연성을 확보.

3. **Spring 통합**:
   - Spring의 다양한 어노테이션 및 컴포넌트를 사용하여 구성 요소를 깔끔하게 관리.

---

#### **6. 개선 제안**

1. **에러 핸들링 추가**:
   - 필터나 AOP 등록 중 예외 발생 시 로깅 및 대체 동작을 명시적으로 정의.
   - 예: `ControllerAdvisorDecorateAspect` 생성 실패 시 로그 출력.

2. **프로퍼티 검증**:
   - `HttpRequestWrapperPoperties`의 `urlPatterns`나 기타 설정값이 잘못된 경우를 대비해 검증 로직 추가.

3. **문서화**:
   - 각 메서드의 목적 및 활성화 조건을 JavaDoc으로 명확히 기술.

4. **테스트 코드 작성**:
   - `ControllerAdvisorDecorateAspect`와 `HttpRequestWrapperFilter`의 동작을 검증하는 단위 및 통합 테스트 추가.

---

#### **7. 통합 테스트 코드 예시**

```java
@SpringBootTest
@AutoConfigureMockMvc
public class ControllerAdvisorConfigTest {

    @Autowired
    private MockMvc mockMvc;

    @Test
    public void testHttpRequestWrapperFilterIsApplied() throws Exception {
        mockMvc.perform(post("/test-endpoint")
                .contentType(MediaType.APPLICATION_JSON)
                .content("{\"key\": \"value\"}"))
               .andExpect(status().isOk())
               .andExpect(result -> {
                   // HttpRequestWrapperFilter가 정상적으로 작동하는지 확인
               });
    }

    @Test
    public void testControllerAdvisorAspect() throws Exception {
        mockMvc.perform(get("/test-endpoint"))
               .andExpect(status().isOk())
               .andExpect(result -> {
                   // ControllerAdvisorDecorateAspect 적용 확인
               });
    }
}
```

---

#### **8. 결론**

`ControllerAdvisorConfig`는 Spring Boot 애플리케이션의 AOP 기반 공통 로직 구현과 요청 본문 처리의 유연성을 제공하는 핵심 구성 클래스입니다. Spring Framework의 다양한 컴포넌트를 적극 활용하며, 구조적 설계와 유연성을 갖추고 있습니다. 위 개선 사항을 적용하면 더욱 안정적이고 유지보수 가능한 구성을 확보할 수 있습니다.
