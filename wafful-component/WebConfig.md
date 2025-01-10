`WebConfig` 클래스는 Spring Boot 애플리케이션에서 **필터, 리스너 및 공통 응답 처리**를 위한 설정을 제공합니다. 특히 CORS, URL 재작성, 요청 로깅, 정적 리소스 처리와 관련된 설정들이 포함되어 있습니다.

---

### **코드 분석**

```java
@AutoConfiguration
@Slf4j
@ConditionalOnWebApplication
@EnableConfigurationProperties({
    CorsFilterProperties.class,
    UrlRewriteFilterProperties.class,
    HttpRequestWrapperPoperties.class,
    RequestLoggingProperties.class,
    StaticResourcesFilterProperties.class
})
public class WebConfig {
```

#### 주요 어노테이션
1. **`@AutoConfiguration`**:
   - Spring Boot에서 자동 설정 클래스로 등록.
2. **`@Slf4j`**:
   - Lombok을 사용하여 로깅(`log` 객체)을 지원.
3. **`@ConditionalOnWebApplication`**:
   - 애플리케이션이 웹 환경에서 실행 중일 때만 활성화.
4. **`@EnableConfigurationProperties`**:
   - 프로퍼티 클래스를 활성화하여 설정 값을 주입.

---

#### **메서드 분석**

##### **1. `corsFilterRegistrationBean`**
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

- **기능**:
  - CORS 필터를 등록하여 특정 도메인에서의 요청을 허용합니다.
  - 동작은 `CorsFilterProperties`에 정의된 값에 따라 설정됩니다.
- **조건**:
  - 프로퍼티 `wafful.cors-filter.enabled`가 `true`일 때만 활성화됩니다.

---

##### **2. `urlRewriteFilterRegistrationBean`**
```java
@Bean
@ConditionalOnProperty(prefix = UrlRewriteFilterProperties.PREFIX, name = "enabled", havingValue = "true")
public FilterRegistrationBean<UrlRewriteFilter> urlRewriteFilterRegistrationBean(UrlRewriteFilterProperties urlRewriteFilterSetting) {
    return new FilterRegistrationBean<>(new UrlRewriteFilter(urlRewriteFilterSetting));
}
```

- **기능**:
  - URL 재작성 필터를 등록하여 요청 URL을 변환합니다.
- **조건**:
  - 프로퍼티 `wafful.url-rewrite-filter.enabled`가 `true`일 때만 활성화됩니다.

---

##### **3. `httpRequestWapperFilterRegistrationBean`**
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

- **기능**:
  - HTTP 요청 본문을 다시 읽을 수 있도록 지원하는 필터를 등록합니다.
- **조건**:
  - 프로퍼티 `wafful.http-request-wrapper-filter.enabled`가 `true`일 때만 활성화됩니다.

---

##### **4. `gtidFilterRegistrationBean`**
```java
@Bean
public FilterRegistrationBean<GtidLogFilter> gtidFilterRegistrationBean() {
    GtidLogFilter filter = new GtidLogFilter();
    FilterRegistrationBean<GtidLogFilter> registrationBean = new FilterRegistrationBean<>(filter);
    registrationBean.addUrlPatterns("/*");
    registrationBean.setOrder(filter.getOrder());
    return registrationBean;
}
```

- **기능**:
  - `GtidLogFilter`를 등록하여 글로벌 트랜잭션 ID를 로깅합니다.
- **조건**:
  - 항상 활성화됩니다.

---

##### **5. `logFilter`**
```java
@Bean
@ConditionalOnProperty(prefix = RequestLoggingProperties.PREFIX, name = "enabled", havingValue = "true")
public CommonsRequestLoggingFilter logFilter(RequestLoggingProperties requestLoggingProperties) {
    CommonsRequestLoggingFilter filter = new CommonLoggingFilter(requestLoggingProperties);
    filter.setBeforeMessagePrefix(requestLoggingProperties.getBeforeMessagePrefix());
    filter.setBeforeMessageSuffix(requestLoggingProperties.getBeforeMessageSuffix());
    filter.setIncludeHeaders(requestLoggingProperties.isIncludeHeaders());
    filter.setIncludeQueryString(requestLoggingProperties.isIncludeQueryString());
    filter.setIncludePayload(requestLoggingProperties.isIncludePayload());
    filter.setIncludeClientInfo(requestLoggingProperties.isIncludeClientInfo());
    filter.setMaxPayloadLength(requestLoggingProperties.getMaxPayloadLength());
    filter.setAfterMessagePrefix(requestLoggingProperties.getAfterMessagePrefix());
    filter.setAfterMessageSuffix(requestLoggingProperties.getAfterMessageSuffix());
    return filter;
}
```

- **기능**:
  - 요청 및 응답 로깅을 위한 필터를 등록합니다.
- **조건**:
  - 프로퍼티 `wafful.common-request-logging-filter.enabled`가 `true`일 때만 활성화됩니다.

---

##### **6. `applicationContextListener`**
```java
@Bean
public ApplicationContextListener applicationContextListener() {
    return new ApplicationContextListener();
}
```

- **기능**:
  - 애플리케이션 컨텍스트 이벤트를 처리하는 리스너를 등록합니다.

---

### **강점**
1. **구성 유연성**:
   - 모든 필터와 리스너는 조건부로 활성화되며, 프로퍼티 값을 통해 제어할 수 있습니다.
2. **확장성**:
   - 신규 필터나 리스너를 추가하기 쉽습니다.
3. **일관성**:
   - 공통적인 설정 방식을 사용하여 코드 가독성이 높습니다.

---

### **개선 제안**
1. **프로퍼티 검증**:
   - 프로퍼티 클래스에 기본값 및 검증 로직을 추가하여 잘못된 설정을 방지할 수 있습니다.
   - 예: `CorsFilterProperties`의 `urlPatterns`가 비어 있을 경우 경고 로그 출력.
2. **테스트 코드**:
   - 각 필터 및 리스너의 동작을 검증하는 통합 테스트를 추가하세요.
3. **문서화**:
   - 주요 설정 항목과 활성화 조건을 문서화하여 유지보수를 용이하게 만드세요.

---

### **통합 테스트 예제**
아래는 `WebConfig` 설정을 테스트하는 코드입니다:

```java
@SpringBootTest
@AutoConfigureMockMvc
public class WebConfigTest {

    @Autowired
    private MockMvc mockMvc;

    @Test
    public void testCorsFilterIsApplied() throws Exception {
        mockMvc.perform(get("/test-endpoint").header("Origin", "http://example.com"))
               .andExpect(header().exists("Access-Control-Allow-Origin"));
    }

    @Test
    public void testLoggingFilter() throws Exception {
        mockMvc.perform(get("/test-endpoint"))
               .andExpect(status().isOk())
               .andExpect(result -> {
                   // 로그 메시지 확인
               });
    }
}
```
