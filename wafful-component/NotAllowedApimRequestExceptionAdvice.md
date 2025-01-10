### 분석 과정:
1. `NotAllowedApimRequestExceptionAdvice`의 구현을 다시 확인하며 `ExceptionAdviceSupporter`와의 상호작용 분석.
2. 두 클래스가 어떻게 협력하여 예외 처리 로직을 구현하는지 심층적으로 설명.

잠시만 기다려 주세요.

### **심층 분석: `ExceptionAdviceSupporter`와 `NotAllowedApimRequestExceptionAdvice`**

---

### **1. `ExceptionAdviceSupporter`와 `NotAllowedApimRequestExceptionAdvice`의 구조**

`ExceptionAdviceSupporter`는 예외 처리 로직의 공통 기능을 정의한 부모 클래스이며,  
`NotAllowedApimRequestExceptionAdvice`는 이를 상속받아 특정 예외(`NotAllowedApimRequestException`)를 처리하는 자식 클래스입니다.

---

### **2. `NotAllowedApimRequestExceptionAdvice` 상세 분석**

`NotAllowedApimRequestExceptionAdvice`는 Spring의 `@ControllerAdvice`를 활용하여 전역적으로 `NotAllowedApimRequestException`을 처리합니다.

#### **주요 기능**
1. **예외 처리**:
   - 발생한 예외를 캡처하여 적절한 HTTP 응답을 생성.
   - 사용자 친화적인 메시지와 상태 코드를 포함.
   
2. **구체적인 응답 생성**:
   - 부모 클래스(`ExceptionAdviceSupporter`)의 로직을 기반으로 헤더, 응답 본문, HTTP 상태 코드를 구성.

---

### **`NotAllowedApimRequestExceptionAdvice` 코드의 주요 메서드**

#### **`handleException`**
`@ExceptionHandler`를 통해 `NotAllowedApimRequestException`을 처리하는 메서드입니다.

```java
@ExceptionHandler({NotAllowedApimRequestException.class})
public ResponseEntity<Object> handleException(NotAllowedApimRequestException e) {

    log.error("@ExceptionHandler(NotAllowedApimRequestException.class) {}", MessageUtil.getStackTraceString(e));

    String code = "03"; // 에러 코드
    String message = e.getMessage(); // 예외 메시지

    HttpHeaders headers = new HttpHeaders();
    headers.add(HTTP_HEADER_BIZ_ERROR, "Y"); // 비즈니스 오류 헤더 추가
    headers.add(SERVER_MESSAGE, ServerMessageUtil.getServerMessageString(code, "not-allowed-apim-request", message));

    ErrorResponseBody body = new ErrorResponseBody(
        INTERNAL_SERVER_ERROR.value(),
        code,
        message,
        applicationName
    );

    return new ResponseEntity<>(body, headers, INTERNAL_SERVER_ERROR);
}
```

#### **핵심 동작**
1. **로그 기록**:
   - 예외 발생 시 스택 트레이스를 로깅합니다.

2. **헤더 생성**:
   - 비즈니스 오류 여부(`Y`)와 서버 메시지를 헤더에 추가.

3. **응답 본문 생성**:
   - `ErrorResponseBody` 객체를 생성하여 HTTP 상태 코드, 에러 코드, 메시지, 애플리케이션 이름을 포함.

4. **HTTP 응답 반환**:
   - `ResponseEntity`를 사용해 응답 헤더와 본문, 상태 코드를 클라이언트로 전달.

---

### **3. 두 클래스 간의 관계와 역할 분담**

#### **`ExceptionAdviceSupporter`의 역할**
- **공통 로직 제공**:
  - 예외 처리 과정에서 반복되는 로직(헤더 구성, 응답 본문 생성, 상태 코드 설정)을 캡슐화.
- **다양한 예외 지원**:
  - `WaffulBusinessException`, `WaffulValidationException` 등 여러 예외 유형을 지원.
- **상속받는 클래스에 유연성을 제공**:
  - 자식 클래스는 특화된 처리만 구현하면 되고, 나머지는 부모 클래스의 메서드를 재사용.

#### **`NotAllowedApimRequestExceptionAdvice`의 역할**
- **특화된 예외 처리**:
  - `NotAllowedApimRequestException` 전용 예외 처리 로직을 구현.
- **클라이언트 응답**:
  - 클라이언트가 문제를 이해할 수 있도록 상세한 메시지와 상태 코드 제공.
- **로그 기록**:
  - 운영 중 디버깅을 위한 정확한 예외 로그 작성.

---

### **4. 클래스 간의 협력**

1. **`NotAllowedApimRequestExceptionAdvice`는 `ExceptionAdviceSupporter`를 상속받아, 부모 클래스의 공통 기능(`getExceptionResponseEntity`)을 활용하여 예외 처리 로직을 단순화합니다.**
   
2. **구체적인 응답 생성 로직(`ErrorResponseBody`, 헤더 구성 등)은 자식 클래스에서 구현하여 특화된 동작을 지원합니다.**

---

### **5. 시나리오 기반 설명**

- **상황**: 클라이언트가 허용되지 않은 API 요청을 보냅니다.
1. 요청 처리 중 `NotAllowedApimRequestException`이 발생합니다.
2. Spring의 `@ExceptionHandler`가 예외를 감지하고, `NotAllowedApimRequestExceptionAdvice`의 `handleException` 메서드가 호출됩니다.
3. `handleException`은 `ExceptionAdviceSupporter`의 공통 로직(`getBizErrorYn`, `getErrorResponseBody`)을 활용하여 응답을 생성합니다.
4. 클라이언트는 에러 메시지와 상태 코드(`500 INTERNAL_SERVER_ERROR`)를 포함한 HTTP 응답을 받습니다.

---

### **결론**

- `ExceptionAdviceSupporter`는 공통 예외 처리 로직을 캡슐화하여 여러 자식 클래스가 재사용할 수 있도록 지원.
- `NotAllowedApimRequestExceptionAdvice`는 이를 상속받아, `NotAllowedApimRequestException`에 특화된 예외 처리 및 응답 생성을 구현.
- 두 클래스는 역할을 명확히 분리하면서도 상호작용하여 코드의 재사용성과 유지보수성을 높입니다.

---

추가적인 질문이나 상세한 설명이 필요하면 언제든 말씀해주세요! 😊
