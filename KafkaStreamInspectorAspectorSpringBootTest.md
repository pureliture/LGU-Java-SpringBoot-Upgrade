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
  - `KafkaStreamInspectorAspector`는 AOP 기반으로 동작하므로, **해당 Aspect가 정상적으로 메시지를 검사하는지 확인할 필요가 있음**.  
  - `MockStaticTestExecutionListener`를 적용하여 **정적 메서드가 예상대로 호출되었는지를 검증하는 기능을 수행함**.  

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
