### `ClientHttpRequestInterceptor`에 대한 상세 설명 (Spring Boot 3.3.5 기준)

---

### 정의 및 역할

`ClientHttpRequestInterceptor`는 Spring Framework의 `org.springframework.http.client` 패키지에 포함된 인터페이스로, Spring에서 제공하는 HTTP 클라이언트(`RestTemplate` 및 `WebClient`)의 요청 처리 과정에서 특정 로직을 삽입하거나 변환할 수 있도록 설계된 컴포넌트입니다.

---

### 주요 기능
1. **요청 변환**:
   - HTTP 요청 헤더, 본문, URL 등을 수정하거나 추가할 수 있습니다.

2. **응답 변환**:
   - HTTP 응답을 처리하기 전에 특정 로직을 삽입하거나 응답 데이터를 변환할 수 있습니다.

3. **체이닝 구조**:
   - 여러 개의 인터셉터를 등록하면 체이닝 방식으로 호출되며, 각 인터셉터는 다음 인터셉터로 요청을 넘기거나 차단할 수 있습니다.

4. **로깅, 보안, 데이터 검증**:
   - 요청/응답 로깅, 인증 토큰 추가, 데이터 검증과 같은 공통 로직을 구현할 수 있습니다.

---

### 주요 메소드

#### `intercept`
```java
ClientHttpResponse intercept(HttpRequest request, byte[] body, ClientHttpRequestExecution execution) throws IOException;
```

**메소드 매개변수**:
- `HttpRequest request`:
  - 요청의 URI, 헤더, 메소드(GET, POST 등)와 같은 정보를 포함하는 객체.
- `byte[] body`:
  - HTTP 요청 본문 데이터.
- `ClientHttpRequestExecution execution`:
  - 요청 실행 체인을 나타내며, 이를 통해 다음 인터셉터 또는 실제 HTTP 요청으로 처리를 위임할 수 있습니다.

**반환값**:
- `ClientHttpResponse`:
  - HTTP 응답 객체로, 요청이 처리된 결과를 포함합니다.

---

### 동작 원리

1. HTTP 요청이 발생하면 `RestTemplate`은 등록된 인터셉터를 차례로 호출합니다.
2. 각 인터셉터는 요청을 수정하거나 특정 작업을 수행할 수 있습니다.
3. 마지막 인터셉터는 요청을 실제 HTTP 클라이언트로 전달하여 응답을 생성합니다.
4. 응답도 인터셉터 체인을 거쳐 반환됩니다.

---

### 사용 예시

#### 1. **요청 로깅 인터셉터**
```java
public class LoggingInterceptor implements ClientHttpRequestInterceptor {

    private static final Logger logger = LoggerFactory.getLogger(LoggingInterceptor.class);

    @Override
    public ClientHttpResponse intercept(HttpRequest request, byte[] body, ClientHttpRequestExecution execution) throws IOException {
        logger.info("Request URI: {}", request.getURI());
        logger.info("Request Method: {}", request.getMethod());
        logger.info("Request Headers: {}", request.getHeaders());

        // 다음 체인으로 요청 전달
        ClientHttpResponse response = execution.execute(request, body);

        logger.info("Response Status Code: {}", response.getStatusCode());
        return response;
    }
}
```

#### 2. **보안 토큰 추가**
```java
public class SecurityInterceptor implements ClientHttpRequestInterceptor {

    private final String token;

    public SecurityInterceptor(String token) {
        this.token = token;
    }

    @Override
    public ClientHttpResponse intercept(HttpRequest request, byte[] body, ClientHttpRequestExecution execution) throws IOException {
        // 요청 헤더에 토큰 추가
        request.getHeaders().add("Authorization", "Bearer " + token);
        return execution.execute(request, body);
    }
}
```

#### 3. **조건부 요청 차단**
```java
public class BlockingInterceptor implements ClientHttpRequestInterceptor {

    @Override
    public ClientHttpResponse intercept(HttpRequest request, byte[] body, ClientHttpRequestExecution execution) throws IOException {
        if (request.getURI().getPath().contains("restricted")) {
            throw new IllegalArgumentException("Access to restricted resources is forbidden.");
        }
        return execution.execute(request, body);
    }
}
```

---

### 등록 및 사용

#### **`RestTemplate`에 등록**
인터셉터는 `RestTemplate`에 설정하여 사용할 수 있습니다.
```java
@Bean
public RestTemplate restTemplate() {
    RestTemplate restTemplate = new RestTemplate();
    restTemplate.getInterceptors().add(new LoggingInterceptor());
    restTemplate.getInterceptors().add(new SecurityInterceptor("your-token-here"));
    return restTemplate;
}
```

#### **`WebClient`와의 차이**
- `RestTemplate`은 `ClientHttpRequestInterceptor`를 통해 요청을 처리하지만, `WebClient`는 `ExchangeFilterFunction`을 사용합니다.
- 비동기 요청을 처리하거나 더 세밀한 제어가 필요한 경우 `WebClient`를 사용할 수 있습니다.

---

### Spring Boot 3.x에서의 변경 사항 및 주의점

1. **Java 17 및 Spring Framework 6.x 호환성**:
   - Spring Boot 3.x는 Java 17과 Spring Framework 6.x 이상에서 작동하므로, 인터셉터 코드도 이를 기준으로 작성해야 합니다.
   - 예를 들어, `HttpRequest`와 같은 인터페이스에서 변경된 메소드를 확인해야 합니다.

2. **`RestTemplate`의 역할 감소**:
   - `RestTemplate`는 여전히 지원되지만, `WebClient`가 추천됩니다. 장기적으로 `WebClient`로의 전환을 고려해야 합니다.

3. **HTTP/2 지원**:
   - Spring Boot 3.x에서 HTTP/2 지원이 강화되었습니다. 인터셉터는 HTTP/2 헤더 처리나 멀티플렉싱 요청의 특성을 고려해야 할 수 있습니다.

---

### 활용 시나리오

1. **로깅 및 모니터링**:
   - 모든 요청과 응답을 기록하여 디버깅 및 모니터링에 활용.

2. **보안 인증 및 검증**:
   - JWT, OAuth 토큰 등의 인증 데이터를 요청 헤더에 추가하거나 검증.

3. **데이터 변경**:
   - 요청 데이터 또는 응답 데이터를 변환하거나 보강.

4. **사용 제한**:
   - 특정 조건에서 요청을 차단하거나, 사용자 맞춤형 예외를 발생.

---

### 설계 사상
- `ClientHttpRequestInterceptor`는 OOP의 **관심사 분리**(Separation of Concerns)를 구현합니다.
- 요청 처리의 공통 기능(로깅, 보안, 데이터 검증)을 분리하여 재사용 가능성을 높이고 유지보수성을 강화합니다.
- 체이닝 구조를 통해 인터셉터 간의 의존성을 제거하고 독립적으로 동작하도록 설계되었습니다.

---

### 요약
- `ClientHttpRequestInterceptor`는 HTTP 요청 및 응답 흐름에서 커스터마이징이 가능하도록 설계된 Spring의 인터페이스입니다.
- 로깅, 인증, 데이터 검증 등 공통 로직을 추가하는 데 적합하며, `RestTemplate`와 통합하여 동작합니다.
- Spring Boot 3.x에서는 여전히 유용하지만, `WebClient`로의 전환도 장기적으로 고려할 필요가 있습니다.
