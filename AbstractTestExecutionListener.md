# **Spring Boot 테스트에서 AbstractTestExecutionListener를 활용한 MockStatic 처리하기**

## **🔍 개요**
Spring Boot 테스트 환경에서는 `@AutoConfiguration` 또는 `@ContextConfiguration`을 통해 자동으로 Bean이 등록되는데,  
일부 Bean이 실행될 때 예상치 못한 API 호출이 발생할 수 있습니다.  
이러한 경우, **Spring의 `AbstractTestExecutionListener`를 활용하여 사전 Mock 처리를 수행하면**  
API 호출을 방지하고 보다 안정적인 테스트 환경을 구성할 수 있습니다.

이번 글에서는 `AbstractTestExecutionListener`의 개념과 활용 방법을 설명한 뒤,  
실제 사용 사례로 `MockStaticTestExecutionListener`을 구현하여 **API 호출을 Mock 처리하는 방법**을 소개하겠습니다. 🚀

---

## **1️⃣ AbstractTestExecutionListener란?**
### **✔️ Spring에서 테스트 실행을 제어하는 리스너**
`AbstractTestExecutionListener`는 Spring의 테스트 실행 라이프사이클에서  
**특정 시점(before/after)에서 커스텀 로직을 실행할 수 있도록 제공되는 리스너 클래스**입니다.

> 📌 **즉, 테스트 실행 전후에 원하는 설정을 추가하거나 정리할 수 있도록 도와주는 기능!**  

### **✔️ 주요 메서드**
| 메서드 | 설명 |
|--------|------|
| `beforeTestClass(TestContext testContext)` | 테스트 클래스 실행 전에 호출됨 |
| `prepareTestInstance(TestContext testContext)` | 테스트 인스턴스가 생성된 후 호출됨 |
| `beforeTestMethod(TestContext testContext)` | 개별 테스트 메서드 실행 전에 호출됨 |
| `afterTestMethod(TestContext testContext)` | 개별 테스트 메서드 실행 후 호출됨 |
| `afterTestClass(TestContext testContext)` | 테스트 클래스 실행이 끝난 후 호출됨 |

### **✔️ AbstractTestExecutionListener를 사용하는 이유**
- **테스트 실행 전후에 특정 설정을 자동으로 적용 가능**  
- **Mock 설정, 리소스 로딩, 데이터 초기화 등을 사전에 수행하여 테스트 안정성 보장**  
- **테스트 후에는 환경을 초기화하여 다른 테스트에 영향을 주지 않도록 함**  

---

## **2️⃣ AbstractTestExecutionListener를 사용해야 하는 상황**
Spring Boot 테스트에서 특정 Bean이 실행될 때,  
**자동으로 API를 호출하는 문제를 해결해야 하는 경우**입니다.

### **✔️ 사용 예제: `@AutoConfiguration`으로 자동 등록된 Bean이 API를 호출하는 경우**
- **문제점:**  
  - `AllowedTopicsReader`는 `afterPropertiesSet()`에서 `PropertyUtil#getResource(String)`을 호출  
  - `PropertyUtil#getResource(String)`는 내부적으로 `Resource#getInputStream()`을 사용하여 실제 API 요청을 수행  
  - API 요청이 Mock 처리되지 않으면, 테스트 실행 중 예외가 발생하여 **정상적인 테스트 진행이 불가능**  
- **해결 방법:**  
  - `AbstractTestExecutionListener`를 활용하여 **테스트 실행 전 API 호출을 Mock 처리**

---

## **3️⃣ 사용 사례: MockStaticTestExecutionListener 구현**
이제, **실제 사용 사례**로 `MockStaticTestExecutionListener`을 구현해 보겠습니다.  
이 리스너는 테스트 실행 전에 **`PropertyUtil#getResource(String)`을 Mock 처리하여**  
`AllowedTopicsReader#afterPropertiesSet()`이 실행되더라도 **실제 API 요청을 발생시키지 않도록 합니다.**

---

### **✔️ 1. `MockStaticTestExecutionListener` 구현**
```java
import com.lguplus.wafful.framework.util.PropertyUtil;
import lombok.SneakyThrows;
import org.mockito.BDDMockito;
import org.mockito.MockedStatic;
import org.mockito.Mockito;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.core.io.ClassPathResource;
import org.springframework.core.io.Resource;
import org.springframework.test.context.TestContext;
import org.springframework.test.context.support.AbstractTestExecutionListener;

import java.io.IOException;
import java.io.InputStream;
import java.nio.charset.StandardCharsets;
import java.util.Scanner;

/**
 * 테스트 실행 전 {@code PropertyUtil#getResource(String)}의 Mocking을 수행하여
 * API 호출을 방지하는 {@link AbstractTestExecutionListener} 구현체입니다.
 */
public class MockStaticTestExecutionListener extends AbstractTestExecutionListener {

    private static MockedStatic<PropertyUtil> mockedStatic;
    private static final Logger log = LoggerFactory.getLogger(MockStaticTestExecutionListener.class);

    private static final String TEST_YAML_NAME = "application.yml";

    @Override
    public void beforeTestClass(TestContext testContext) {
        Mockito.framework().clearInlineMocks();
        mockedStatic = Mockito.mockStatic(PropertyUtil.class);

        String publishResourceUrl = this.getTargetUrlFromProperty(TEST_YAML_NAME,
                "wafful.inspector.topic.publish.resource-url");
        String subscribeResourceUrl = this.getTargetUrlFromProperty(TEST_YAML_NAME,
                "wafful.inspector.topic.subscribe.resource-url");

        this.registerMockResourceForTargetUrl(publishResourceUrl, loadMockDataFromFile("mock-data/publish-messages.json"));
        this.registerMockResourceForTargetUrl(subscribeResourceUrl, loadMockDataFromFile("mock-data/subscribe-messages.json"));
    }

    @Override
    public void afterTestClass(TestContext testContext) {
        if (mockedStatic != null) {
            mockedStatic.close();
            mockedStatic = null;
        }
        Mockito.framework().clearInlineMocks();
    }

    private String getTargetUrlFromProperty(String ymlPath, String propertyName) {
        return "http://mocked-api.com/" + propertyName;
    }

    @SneakyThrows
    private void registerMockResourceForTargetUrl(String targetUrl, String intendedResponse) {
        Resource mockResource = Mockito.mock(Resource.class);
        InputStream fakeInputStream = new ByteArrayInputStream(intendedResponse.getBytes(StandardCharsets.UTF_8));
        BDDMockito.given(mockResource.getInputStream()).willReturn(fakeInputStream);

        mockedStatic.when(() -> PropertyUtil.getResource(targetUrl)).thenReturn(mockResource);
        log.info("✅ Mock Resource 등록 완료: {}", targetUrl);
    }

    private String loadMockDataFromFile(String resourcePath) {
        try {
            Resource resource = new ClassPathResource(resourcePath);
            InputStream inputStream = resource.getInputStream();
            Scanner scanner = new Scanner(inputStream, StandardCharsets.UTF_8).useDelimiter("\\A");
            return scanner.hasNext() ? scanner.next() : "";
        } catch (IOException e) {
            log.error("❌ 리소스 파일을 읽는 중 오류 발생: {}", resourcePath, e);
            throw new RuntimeException("Mock 데이터 로드 실패: " + resourcePath, e);
        }
    }
}
```

---

### **✔️ 2. `MockStaticTestExecutionListener` 적용**
```java
@TestExecutionListeners(value = MockStaticTestExecutionListener.class, mergeMode = MERGE_WITH_DEFAULTS)
@SpringBootTest
class KafkaStreamInspectorAspectSpringBootTest {
  
    @Test
    void testWithoutRealAPI() {
        // MockedStatic이 적용된 상태에서 테스트 실행
    }
}
```

---

## **4️⃣ MockStaticTestExecutionListener를 활용한 기대 효과**
✅ **테스트 실행 전 PropertyUtil의 API 호출을 Mock 처리하여 불필요한 API 요청 방지**  
✅ **`AllowedTopicsReader#afterPropertiesSet()` 실행 중 API 호출로 인한 테스트 실패 방지**  
✅ **Spring Boot 실행 시 자동으로 API 요청이 발생하는 문제 해결**  
✅ **Mock 데이터를 `src/test/resources/` 내 JSON 파일로 관리하여 유지보수성 향상**  

---

## **🎯 결론**
- Spring Boot 테스트에서는 예상치 못한 **자동 API 호출이 문제를 일으킬 수 있음**  
- **`AbstractTestExecutionListener`를 활용하면 테스트 실행 전 특정 설정을 적용 가능**  
- `MockStaticTestExecutionListener`를 사용하면 **테스트 환경에서 안전하게 API 호출을 Mock 처리 가능**  
