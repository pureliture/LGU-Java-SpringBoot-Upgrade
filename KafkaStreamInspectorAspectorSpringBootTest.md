## Rev1
### **KafkaStreamInspectorAspector 검증 전략 및 구현 보고서**  

---

## **1. 개요**  
본 문서는 `KafkaStreamInspectorAspectorSpringBootTest`를 활용하여 `KafkaStreamInspectorAspector`의 동작을 검증하는 방식에 대한 내용을 포함함.  
Spring Cloud Stream 환경에서 Kafka 메시지 송수신을 제어하는 `KafkaStreamInspectorAspector`가 정상적으로 동작하는지를 검증하기 위해 **바인딩을 변경하고 예외 처리 로직을 테스트하는 방식**을 채택함.  

이를 위해 `MockStaticTestExecutionListener`, `SpringCloudStreamTestConfig`, `SpringCloudStreamBindModifier` 등의 테스트 유틸리티를 조합하여 테스트 환경을 구성하고 검증을 수행함.

---

## **2. KafkaStreamInspectorAspector 검증 방식**  

`KafkaStreamInspectorAspector`는 Spring AOP 기반으로 동작하며, Kafka 메시지를 감시하고 정책에 맞는 메시지인지 검증하는 역할을 수행함.  
테스트에서는 **Kafka 바인딩을 변경하고 메시지 송수신 여부를 확인하는 방식**을 사용하여 검증을 진행함.

---

## **3. 검증 환경 구성**  

`KafkaStreamInspectorAspectorSpringBootTest`는 단순한 단위 테스트가 아니라, **Spring Boot Test 컨텍스트를 활용한 통합 테스트 방식**을 따름.  
이를 위해 다음과 같은 유틸리티 클래스들을 조합하여 검증 환경을 구성함.

### **3.1 MockStaticTestExecutionListener**
- **역할**: 특정 정적 메서드 호출을 감지하고, 필요 시 Mocking을 수행하는 기능을 제공함.  
- **사용된 이유**:  
  - Application Context 생성 도중에 `AllowedTopicsReader`에서 비동기메시징 포털 토픽 목록 조회 API를 호출함
    - `PropertyUtil.getResource`로 `Resource` 객체를 생성하여 API 호출
  - 테스트 코드에서는 실제 API 호출이 발생하지 않게 API 호출을 Mock 처리해야함
  - `MockStaticTestExecutionListener`에서 Application Context 생성이 시작되기 전에 `MockedStatic`으로 `PropertyUtil.getResource` 메소드를 Mock 처리하여 실제 API 호출이 발생하지 않게 방지

### **3.2 SpringCloudStreamTestConfig**
- **역할**: Spring Cloud Stream의 테스트 환경을 구성하는 역할을 수행함.  
- **사용된 이유**:  
  - Spring Cloud Stream을 활용하여 Kafka 메시징 환경을 테스트하기 위해 필요함.  
  - `KafkaStreamInspectorAspectorSpringBootTest`에서는 **Kafka 브로커를 직접 실행하는 것이 아니라, Spring Cloud Stream의 가상 바인딩을 이용하여 검증하는 방식**을 채택함.  
  - `SpringCloudStreamTestConfig`를 이용하여 **테스트 실행 시 메시징 관련 Bean들을 자동으로 설정하고, Spring 컨텍스트 내에서 정상적으로 동작할 수 있도록 구성함**.  

### **3.3 SpringCloudStreamBindModifier**
- **역할**: `BindingService`를 활용하여 **Kafka 바인딩(destination)을 동적으로 변경하고, 초기 상태로 복원하는 기능을 수행함**.  
- **사용된 이유**:  
  - `KafkaStreamInspectorAspectorSpringBootTest`에서는 **각 테스트 실행 시 Kafka 바인딩을 변경하여 메시지가 올바르게 송수신되는지 확인하는 방식으로 검증을 수행함**.  
  - 바인딩 변경 후 다른 테스트에 영향을 주지 않도록, 매 테스트 실행 전에 `initializeBindingService()`를 호출하여 바인딩을 초기화함.  

---

## **4. KafkaStreamInspectorAspector 검증 과정**  

테스트에서는 **Kafka 바인딩을 동적으로 변경한 후, 메시지 송수신 여부와 예외 처리 로직을 검증하는 방식**을 사용함.

### **4.1 테스트 실행 전 초기화**
- `@BeforeEach`를 사용하여 `initializeBindingService()`를 호출함.  
- 이전 테스트에서 변경된 Kafka 바인딩을 초기 상태로 복원하는 역할을 수행함.  

### **4.2 바인딩 변경 후 메시지 송수신 검증**
- `changeProducerFunctionBindingsDestination()`을 호출하여 특정 `Producer`의 바인딩(destination)을 변경함.  
- `BindingService`를 활용하여 destination이 정상적으로 변경되었는지 검증하는 과정이 포함됨.  

- `changeConsumerFunctionBindingsDestination()`을 호출하여 특정 `Consumer`의 바인딩(destination)을 변경함.  
- 변경된 바인딩을 통해 메시지가 정상적으로 수신되는지 확인하는 과정이 포함됨.  

### **4.3 예외 처리 검증**
- 등록되지 않은 토픽을 사용하면 `NotRegisteredPublishException`이 발생하는지 확인함.  
- 이를 통해 **KafkaStreamInspectorAspector가 정의된 정책에 따라 예외를 정상적으로 발생시키는지 검증함**.  

---

## **5. KafkaStreamInspectorAspectorSpringBootTest의 검증 방식 요약**  

1. **테스트 실행 전 초기화**  
   - `SpringCloudStreamBindModifier.initializeBindingService()`를 호출하여 **Kafka 바인딩을 초기 상태로 복원하는 과정 포함**  
2. **Kafka 바인딩 변경 후 메시지 송수신 검증**  
   - `changeProducerFunctionBindingsDestination()` 및 `changeConsumerFunctionBindingsDestination()`을 활용하여 **바인딩을 변경하고 메시지가 정상적으로 송수신되는지 확인하는 과정 포함**  
3. **예외 발생 검증**  
   - 정의되지 않은 토픽으로 메시지를 보낼 경우 `NotRegisteredPublishException`이 발생하는지 확인하는 과정 포함  

---

## **6. KafkaStreamInspectorAspector 검증을 위한 통합 테스트 방식 채택 이유**  

- 단순한 단위 테스트가 아니라, **실제 Spring Boot 컨텍스트 내에서 동작을 검증하기 위해 통합 테스트 방식을 적용함**.  
- `SpringCloudStreamTestConfig`, `SpringCloudStreamBindModifier`를 조합하여, **실제 Kafka 환경을 구성하지 않고도 메시지 송수신을 검증할 수 있도록 설계됨**.  
- `MockStaticTestExecutionListener`를 활용하여, **AOP 기반으로 동작하는 KafkaStreamInspectorAspector의 메서드가 정상적으로 호출되는지를 확인하는 과정 포함됨**.  

---

## **7. 향후 개선 방향**  

- 현재 **Spring Cloud Stream을 활용한 가상 환경에서만 검증이 이루어짐**  
  → **실제 Kafka 브로커 환경에서도 동일한 테스트를 수행하는 E2E 테스트 추가 필요**  
- AOP의 성능 테스트 추가 필요  
  → 현재는 메시지 필터링이 정상적으로 동작하는지만 확인하지만, **실제 대량의 메시지가 흐를 때 성능을 측정하는 추가 테스트 필요**  

---

## **8. 결론**  

`KafkaStreamInspectorAspectorSpringBootTest`는 Spring Boot Test 환경에서 **KafkaStreamInspectorAspector가 정상적으로 메시지를 검증하는지 확인하는 통합 테스트**를 수행하는 방식으로 작성됨.  
이를 위해 `SpringCloudStreamBindModifier`를 활용하여 동적으로 바인딩을 변경하고, `MockStaticTestExecutionListener`를 사용하여 AOP 기반 메서드가 실행되는지 확인하는 과정을 포함함.  

이러한 접근 방식은 **실제 Kafka 브로커를 실행하지 않고도 Spring Cloud Stream을 활용하여 통합 테스트를 수행할 수 있도록 설계된 방식이며**, 검증 프로세스를 자동화하여 신뢰성을 높이는 데 기여함. 🚀


## Rev2
# **Spring Cloud Stream & Eventuate AOP 유효성 검사 테스트 코드 추가**  

> **프로젝트에서 활용 중이던 Spring Cloud Stream 및 Eventuate의 AOP 기반 메시지 유효성 검사 기능에 대한 테스트 코드를 보완했습니다.**  
> **Eventuate는 Spring에서 공식적으로 제공하는 기능이 아니기 때문에, 해당 AOP 기능에 대한 검증은 Mockito를 활용한 단위 테스트로 대체했습니다.**  

---

## **🎯 테스트 코드 보완이 필요한 이유**  
Spring Cloud Stream 및 Eventuate 기반의 메시지 전송 시, **허용되지 않은 토픽이나 이벤트 대상으로 메시지가 전송되는 경우 이를 차단하는 AOP 기반 유효성 검사 기능을 활용하고 있습니다.**  
하지만 기존 코드에서는 **이 기능이 정상적으로 동작하는지 검증하는 테스트 코드가 부족**했기 때문에, 이를 보완하여 **보다 안정적인 테스트 환경을 구축**하고자 했습니다.  

특히, **Spring Cloud Stream은 Spring Boot에서 공식적으로 지원하는 기능이므로, 이에 대한 테스트 코드를 직접 작성할 수 있었습니다.**  
반면, **Eventuate는 공식적인 Spring 지원 기능이 아니기 때문에 레퍼런스가 부족**했으며,  
이에 따라 **Eventuate AOP 기능에 대한 테스트는 Mockito를 활용한 단위 테스트로 대체했습니다.**  

---

## **✅ 테스트 코드 보완을 위해 해결해야 했던 문제들**  

### **1️⃣ Eventuate AutoConfiguration 충돌 문제**  
Spring Boot 테스트 환경에서 **Eventuate AutoConfiguration이 자동으로 로드되면서 충돌이 발생**하는 문제가 있었습니다.  
만약 Eventuate 관련 AutoConfiguration이 포함된 상태로 테스트를 실행하면,  
아래와 같은 **Bean 생성 예외(`BeanCreationException`)**가 발생했습니다:  

```
Exception encountered during context initialization - cancelling refresh attempt:
org.springframework.beans.factory.BeanCreationException: Error creating bean with name 'eventuateCommonJdbcOperations'
defined in class path resource [io/eventuate/common/spring/jdbc/EventuateCommonJdbcOperationsConfiguration.class]: 
Unexpected exception during bean creation
```

**💡 해결 방법**  
- Eventuate의 AutoConfiguration을 **개별적으로 `@EnableAutoConfiguration(exclude = ...)`로 제외하기에는 너무 많은 설정이 필요**했습니다.  
- 따라서 **패키지 단위(`io.eventuate` 전체)를 대상으로 AutoConfiguration을 필터링**하여 일괄적으로 제외했습니다.  

---

### **2️⃣ Spring Cloud Stream 바인딩 테스트를 위한 프로파일 기반 테스트 환경 구축**  
Spring Boot의 기본 동작에 따라, **profile이 `default`가 아닌 경우 `application-{profile}.yml` 파일을 참조**합니다.  
이를 활용하여 **테스트 환경에서 `application-test.yml`을 로드하도록 설정**했습니다.  

→ `@ActiveProfiles("test")`, `@EnabledIfSystemProperty`를 활용하여 테스트 환경을 설정했습니다.  

---

### **3️⃣ MockStatic을 활용한 정적 메서드 모킹 (HTTP 요청 차단 목적)**  
AOP에서 메시지의 허용 여부를 검사할 때, **허용된 메시지인지 확인하기 위한 기준 데이터를 API를 통해 조회하는 과정이 포함**되어 있었습니다.  
이 API는 **내부적으로 `Resource`를 생성하여 Http 요청을 수행하는 구조**였으며,  
테스트 실행 중 **불필요한 HTTP 요청이 발생하는 문제**가 있었습니다.  

이를 해결하기 위해 **해당 Resource를 생성하는 Static 메서드를 Mock 처리하여, 실제 Http 요청이 발생하지 않도록 조치**했습니다.  

---

## **🚀 문제 해결을 위해 도입한 기술**  

### **1️⃣ Eventuate AutoConfiguration 제외를 위한 패키지 기반 필터링**  
#### **🔍 특정 AutoConfiguration 클래스를 제외하는 방식이 아닌 패키지 단위로 제외한 이유**  
Eventuate의 AutoConfiguration 클래스들은 **개별적으로 `exclude`하기에는 너무 많고 의존성이 복잡**했습니다.  
따라서 **패키지 패스(`io.eventuate`)를 기반으로 제외하는 AutoConfiguration 필터를 추가**했습니다.  

```java
public class AutoConfigurationImportFilterForSpringBootTest implements AutoConfigurationImportFilter {
    private static final List<String> EXCLUDED_PACKAGES = Arrays.asList("io.eventuate");

    @Override
    public boolean[] match(String[] autoConfigurationClasses, AutoConfigurationMetadata autoConfigurationMetadata) {
        boolean[] matchResults = new boolean[autoConfigurationClasses.length];
        for (int i = 0; i < autoConfigurationClasses.length; i++) {
            matchResults[i] = autoConfigurationClasses[i] == null || 
                              EXCLUDED_PACKAGES.stream().noneMatch(autoConfigurationClasses[i]::startsWith);
        }
        return matchResults;
    }
}
```

✅ **적용 결과**  
- `io.eventuate` 패키지 아래의 AutoConfiguration을 **테스트 환경에서 제외**했습니다.  
- Eventuate의 AutoConfiguration에서 발생하는 **Bean 생성 예외(`BeanCreationException`) 문제를 해결**했습니다.  
- Spring Cloud Stream 테스트 환경이 **정상적으로 실행될 수 있도록 보장**했습니다.  

---

### **2️⃣ MockStatic을 활용한 정적 메서드 모킹 (HTTP 요청 차단)**  
테스트 환경에서 **정적 메서드 호출(MockStatic)을 모킹**하여 HTTP 요청이 발생하지 않도록 처리했습니다.  
이 방식으로, **Resource 객체를 생성하는 과정에서 API 호출이 발생하는 문제를 해결**할 수 있었습니다.  

```java
public class MockStaticTestExecutionListener extends AbstractTestExecutionListener {
    private static MockedStatic<PropertyUtil> mockedPropertyUtil;

    @Override
    public void beforeTestClass(TestContext testContext) {
        mockedPropertyUtil = Mockito.mockStatic(PropertyUtil.class);
        mockedPropertyUtil.when(() -> PropertyUtil.getResource(Mockito.anyString()))
                          .thenAnswer(invocation -> {
                              String url = invocation.getArgument(0);
                              return new ClassPathResource("mock/" + url);
                          });
    }

    @Override
    public void afterTestClass(TestContext testContext) {
        if (mockedPropertyUtil != null) mockedPropertyUtil.close();
    }
}
```

✅ **적용 결과**  
- **테스트 실행 중 HTTP 요청이 발생하지 않도록 차단**했습니다.  
- **Resource 객체를 생성하는 과정에서 실제 네트워크 통신이 이루어지지 않도록 조치**했습니다.  
- **테스트가 보다 빠르고 안정적으로 실행될 수 있도록 개선**했습니다.  

---

### **3️⃣ 프로파일 기반 동적 테스트 실행**  
Kafka Stream 바인딩 테스트가 **특정 프로파일에서만 실행되도록 설정**했습니다.  
→ `@ActiveProfiles("test")`, `@EnabledIfSystemProperty`를 활용하여 프로파일을 설정했습니다.  

```java
@SpringBootTest
@ActiveProfiles("test")
@EnabledIfSystemProperty(named = "spring.profiles.active", matches = "test")
class KafkaStreamInspectorAspectSpringBootTest {
    @Test
    void testBindProducer_givenNotAllowedTopic_whenBinding_thenThrowException() {
        assertThrows(NotRegisteredPublishException.class, () -> {
            springCloudStreamBindModifier.changeProducerFunctionBindingsDestination(
                "publish", SpringCloudStreamBindModifier.BindingChannel.OUT, 0, "unregistered_topic");
        });
    }
}
```

✅ **적용 결과**  
- Spring Cloud Stream AOP 테스트를 **운영 환경과 격리**했습니다.  
- **테스트가 특정 프로파일(`test`)에서만 실행되도록 보장**했습니다.  
- **Eventuate AOP는 테스트 대상이 아님을 명확하게 구분**했습니다.  

---

## **🔍 결론**  
### **✅ AOP 기반 메시지 유효성 검사를 테스트하기 위해 해결한 문제들**  
| 문제 | 해결 방법 |
|------|---------|
| **Eventuate AutoConfiguration 로딩 충돌** | `AutoConfigurationImportFilterForSpringBootTest` 활용하여 `io.eventuate` 패키지 제외 |
| **테스트 중 불필요한 HTTP 요청 발생** | `MockStaticTestExecutionListener`로 `PropertyUtil.getResource()` Mock 처리 |
| **Kafka 바인딩 변경 테스트를 격리** | `@ActiveProfiles("test")` 및 `@EnabledIfSystemProperty` 사용 |

이번 테스트 코드 추가를 통해, **AOP 기반 유효성 검사 기능이 정상적으로 동작함을 검증**하고,  
**테스트 환경에서 불필요한 AutoConfiguration 문제를 해결하여 Spring Cloud Stream 테스트를 안정적으로 수행할 수 있었습니다.** 🚀
