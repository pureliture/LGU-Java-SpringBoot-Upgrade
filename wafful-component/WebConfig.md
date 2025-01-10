### **보고서: WebConfig 클래스 분석**

---

#### **1. 클래스 개요**

`WebConfig`는 Spring Boot 애플리케이션에서 요청 처리, 필터링, CORS, URL 재작성, 정적 리소스 핸들링, 요청 로깅 등을 설정하는 중요한 구성 클래스입니다. 이 클래스는 다양한 Spring Framework 컴포넌트와 통합되어 웹 애플리케이션의 유연성과 확장성을 보장합니다.

---

#### **2. 주요 어노테이션 및 역할**

- **`@AutoConfiguration`**:
  - Spring Boot의 자동 구성 클래스로 등록됩니다.
  - Spring Boot가 실행 시 이 클래스의 설정을 로드합니다.

- **`@Slf4j`**:
  - Lombok을 사용하여 로깅 기능을 제공합니다.

- **`@ConditionalOnWebApplication`**:
  - 애플리케이션이 웹 환경일 때만 이 클래스가 활성화됩니다.

- **`@EnableConfigurationProperties`**:
  - `CorsFilterProperties`, `UrlRewriteFilterProperties` 등 여러 프로퍼티 클래스를 활성화하여 Spring Boot의 `application.yml` 또는 `application.properties` 파일에서 설정값을 읽어올 수 있게 합니다.

---

#### **3. 클래스에서 사용된 Spring Framework 컴포넌트**

| Spring 컴포넌트                  | 역할 및 설명                                                                                  |
|----------------------------------|---------------------------------------------------------------------------------------------|
| `FilterRegistrationBean`        | 서블릿 필터를 등록하고, URL 패턴 및 우선순위를 설정합니다.                                     |
| `ServletListenerRegistrationBean` | 서블릿 리스너를 등록하여 요청 이벤트를 처리합니다.                                              |
| `CommonsRequestLoggingFilter`   | HTTP 요청 및 응답 정보를 로깅할 수 있는 기본 제공 필터입니다.                                    |
| `@ConditionalOnProperty`        | 특정 프로퍼티 조건이 충족될 때만 Bean을 생성합니다.                                             |
| `Environment`                   | 애플리케이션의 환경 정보를 제공하며, 프로퍼티 값 및 프로파일 관련 작업에 사용됩니다.              |
| `@Bean`                         | Spring 컨텍스트에 빈을 등록하여 다른 컴포넌트에서 의존성을 주입받을 수 있게 합니다.              |

---

#### **4. 주요 메서드 분석**

##### **4.1 CORS 필터 등록**
```java
@Bean
@ConditionalOnProperty(prefix = CorsFilterProperties.PREFIX, name = "enabled", havingValue = "true")
public FilterRegistrationBean<CorsFilter> corsFilterRegistrationBean(CorsFilterProperties corsFilterSetting) {
    String allowOrigin = corsFilterSetting.getAccessControlAllowOrigin();
    boolean allowCredentials = corsFilterSetting.isAccessControlAllowCredentials();
    String allowHeaders = corsFilterSetting.getAccessControlAllowHeaders();
    String allowMethods = corsFilterSetting.getAccessControlAllowMethods();
    int maxAge = corsFilterSetting.getAccessControlMaxAge();

    FilterRegistrationBean<CorsFilter> registrationBean = new FilterRegistrationBean<>(
        new CorsFilter(allowOrigin, allowCredentials, allowMethods, allowHeaders, maxAge)
    );
    registrationBean.addUrlPatterns(corsFilterSetting.getUrlPatterns());
    log.info("wafful cors-filter enabled!!!");
    return registrationBean;
}
```

- **설명**:
  - `FilterRegistrationBean`을 사용하여 CORS 필터를 등록합니다.
  - 허용된 Origin, Header, Methods 등을 `CorsFilterProperties`에서 읽어 설정합니다.
- **Spring 컴포넌트**:
  - `FilterRegistrationBean`: 필터 등록을 담당.
  - `@ConditionalOnProperty`: 특정 프로퍼티 값이 충족될 때만 필터를 활성화.

---

##### **4.2 URL 재작성 필터 등록**
```java
@Bean
@ConditionalOnProperty(prefix = UrlRewriteFilterProperties.PREFIX, name = "enabled", havingValue = "true")
public FilterRegistrationBean<UrlRewriteFilter> urlRewriteFilterRegistrationBean(UrlRewriteFilterProperties urlRewriteFilterSetting) {
    return new FilterRegistrationBean<>(new UrlRewriteFilter(urlRewriteFilterSetting));
}
```

- **설명**:
  - 요청 URL을 변환하는 `UrlRewriteFilter`를 등록합니다.
  - `FilterRegistrationBean`을 사용해 필터를 설정하고, `UrlRewriteFilterProperties`에서 값을 읽습니다.
- **Spring 컴포넌트**:
  - `FilterRegistrationBean`: URL 재작성 로직을 필터로 등록.

---

##### **4.3 HTTP 요청 래퍼 필터 등록**
```java
@Bean
@ConditionalOnProperty(prefix = HttpRequestWrapperPoperties.PREFIX, name = "enabled", havingValue = "true")
public FilterRegistrationBean<HttpRequestWrapperFilter> httpRequestWapperFilterRegistrationBean(HttpRequestWrapperPoperties httpRequestWrapperSetting) {
    HttpRequestWrapperFilter filter = new HttpRequestWrapperFilter();
    FilterRegistrationBean<HttpRequestWrapperFilter> registrationBean = new FilterRegistrationBean<>(filter);
    registrationBean.addUrlPatterns(httpRequestWrapperSetting.getUrlPatterns());
    registrationBean.setOrder(filter.getOrder());
    return registrationBean;
}
```

- **설명**:
  - 요청 본문(`RequestBody`)을 여러 번 읽을 수 있도록 `HttpRequestWrapperFilter`를 등록합니다.
- **Spring 컴포넌트**:
  - `FilterRegistrationBean`: 요청 래퍼 필터 등록.

---

##### **4.4 요청 로깅 필터 등록**
```java
@Bean
@ConditionalOnProperty(prefix = RequestLoggingProperties.PREFIX, name = "enabled", havingValue = "true")
public CommonsRequestLoggingFilter logFilter(RequestLoggingProperties requestLoggingProperties) {
    CommonsRequestLoggingFilter filter = new CommonLoggingFilter(requestLoggingProperties);
    filter.setBeforeMessagePrefix(requestLoggingProperties.getBeforeMessagePrefix());
    filter.setIncludeHeaders(requestLoggingProperties.isIncludeHeaders());
    return filter;
}
```

- **설명**:
  - `CommonsRequestLoggingFilter`를 등록하여 요청 정보를 로깅합니다.
  - 프로퍼티 설정값에 따라 로깅 동작을 제어합니다.
- **Spring 컴포넌트**:
  - `CommonsRequestLoggingFilter`: Spring이 제공하는 로깅 필터.

---

#### **5. 강점**
1. **유연한 구성**:
   - 각 설정은 프로퍼티 값을 기반으로 조건부로 활성화됩니다.
   - `@EnableConfigurationProperties`를 통해 프로퍼티 설정과 Java 코드 간 연계성을 강화.
2. **확장성**:
   - 신규 필터나 리스너를 쉽게 추가할 수 있는 구조.
3. **Spring 통합**:
   - Spring Framework의 컴포넌트를 적극 활용하여 유지보수성과 코드 간결성을 확보.

---

#### **6. 개선 제안**
1. **프로퍼티 검증**:
   - 프로퍼티 값의 유효성을 검증하는 로직 추가.
   - 예: `CorsFilterProperties`에서 `urlPatterns`가 비어 있으면 경고 로그 출력.
2. **테스트 코드 작성**:
   - 각 필터 및 리스너가 예상대로 동작하는지 확인하기 위한 통합 테스트 추가.
3. **문서화**:
   - 주요 설정 및 활성화 조건을 정리한 개발자 문서를 제공.

---

#### **7. 통합 테스트 코드 예시**

```java
@SpringBootTest
@AutoConfigureMockMvc
public class WebConfigTest {

    @Autowired
    private MockMvc mockMvc;

    @Test
    public void testCorsFilterIsApplied() throws Exception {
        mockMvc.perform(get("/test-endpoint").header("Origin", "http://example.com"))
               .andExpect(header().string("Access-Control-Allow-Origin", "http://example.com"));
    }

    @Test
    public void testLoggingFilter() throws Exception {
        mockMvc.perform(get("/test-endpoint"))
               .andExpect(status().isOk())
               .andExpect(result -> {
                   // 요청 및 응답 로깅 확인
               });
    }
}
```

---

#### **8. 결론**
`WebConfig` 클래스는 Spring Boot 애플리케이션에서 필터, 리스너, 공통 처리 로직을 설정하는 핵심 구성 클래스입니다. Spring Framework 컴포넌트를 효과적으로 활용하며 유연성과 확장성을 제공합니다. 위 제안된 개선 사항을 통해 더욱 안정적이고 유지보수 가능한 구성을 구축할 수 있습니다.
