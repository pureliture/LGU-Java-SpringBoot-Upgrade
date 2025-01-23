### @EmbeddedKafka 사용 시 @AutoConfiguration 적용 유무에 따른 구성 비교

---

#### **1. 배경**

Spring Kafka의 **`@EmbeddedKafka`**는 테스트 환경에서 **로컬 Kafka 브로커**를 실행하여 메시지 송신 및 수신 동작을 검증할 수 있도록 해주는 기능입니다.  
테스트 환경을 구성할 때, `@EmbeddedKafka`를 직접 사용하거나, Spring의 **`@AutoConfiguration`**을 통해 설정을 자동화할 수 있습니다.  

이 글에서는 **직접 구성 방식**과 **자동 구성 방식**의 차이점, `@AutoConfiguration`이 자동으로 추가해주는 구성 요소, 그리고 각 방식의 장단점을 비교합니다.

---

#### **2. 직접 구성 방식 (`@ContextConfiguration`)**

##### **구성 방법**

`@EmbeddedKafka`를 사용하면서 필요한 설정만 명시적으로 정의했습니다.  
**KafkaAutoConfiguration**을 `@ContextConfiguration`으로 추가하여 Kafka 관련 Bean만 로드되도록 설정했습니다.

```java
@ContextConfiguration(classes = {KafkaAutoConfiguration.class, TopicInspectorTestConfig.class, KafkaStreamInspectorTestConfig.class})
@SpringBootTest
@EmbeddedKafka(partitions = 1, topics = {"test-topic"})
@DirtiesContext(classMode = DirtiesContext.ClassMode.AFTER_CLASS)
public class KafkaStreamInspectorAspectorSpringBootTest { ... }
```

##### **직접 추가한 구성 요소**
1. **KafkaAutoConfiguration**: Kafka 관련 기본 Bean 설정을 로드.
2. **`@DirtiesContext`**: 테스트 리소스 충돌 방지.
3. **`application.yml`**: Telemetry, Producer, Consumer 등 테스트에 필요한 설정 명시.

##### **주요 특징**
- 필요한 구성 요소만 수동으로 설정하며, **최소한의 리소스 사용**.
- Spring Boot의 자동 구성을 활용하지 않아 **컨텍스트 로드 속도 최적화**.

---

#### **3. 자동 구성 방식 (`@AutoConfiguration`)**

Spring Boot의 **`@AutoConfiguration`**은 Kafka 및 Embedded Kafka를 사용하기 위한 구성 요소를 자동으로 등록합니다.  
아래는 Spring Boot 3.x와 Spring Kafka 3.x 기준으로 `@AutoConfiguration`이 추가해주는 주요 구성 요소입니다.

---

##### **3.1. 자동 추가되는 구성 요소**

| **Bean**                         | **설명**                                                                                                  |
|----------------------------------|----------------------------------------------------------------------------------------------------------|
| **EmbeddedKafkaBroker**          | 테스트 브로커 인스턴스를 생성. 파티션 및 토픽 구성을 자동으로 처리.                                              |
| **KafkaAdmin**                   | Kafka 토픽 관리(생성, 삭제 등)를 위한 Admin 클라이언트 생성.                                                  |
| **ProducerFactory**              | Kafka 메시지 송신을 위한 Producer 인스턴스를 생성.                                                              |
| **ConsumerFactory**              | Kafka 메시지 수신을 위한 Consumer 인스턴스를 생성.                                                              |
| **KafkaTemplate**                | Kafka 메시지 송신을 위한 템플릿.                                                                              |
| **ConcurrentKafkaListenerContainerFactory** | Kafka Listener 등록 및 처리. Consumer 메시지를 비동기로 처리할 수 있도록 설정.                                    |
| **KafkaListenerEndpointRegistry**| 모든 Kafka Listener를 관리하고 상태를 제어할 수 있는 Registry.                                                 |

---

##### **3.2. 자동 구성에서 사용하는 프로퍼티**

`@AutoConfiguration`은 `application.yml` 또는 `application.properties`에서 설정된 프로퍼티를 자동으로 적용합니다.  
기본적으로 아래와 같은 설정이 적용됩니다:

| **프로퍼티**                       | **설명**                                                                                                  | **기본값**          |
|----------------------------------|----------------------------------------------------------------------------------------------------------|---------------------|
| `spring.kafka.bootstrap-servers` | Kafka 브로커 주소. Embedded Kafka를 사용하는 경우 자동으로 `embeddedKafkaBroker` 주소를 바인딩.              | -                   |
| `spring.kafka.consumer.*`        | Consumer 관련 설정(역직렬화, 오프셋, 그룹 ID 등).                                                          | 기본값 사용         |
| `spring.kafka.producer.*`        | Producer 관련 설정(직렬화, acks, 배치 크기 등).                                                            | 기본값 사용         |
| `spring.kafka.admin.*`           | KafkaAdmin 관련 설정(토픽 자동 생성 여부 등).                                                              | 기본값 사용         |

---

#### **4. 직접 구성 방식 vs 자동 구성 방식 비교**

| **항목**                        | **직접 구성 방식**                                                                                      | **자동 구성 방식**                                                                                   |
|--------------------------------|------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| **구성 방식**                   | 필요한 Bean을 명시적으로 정의하고, 최소한의 설정만 로드.                                                       | `@AutoConfiguration`을 통해 모든 Kafka 관련 Bean과 설정을 자동으로 로드.                                      |
| **컨텍스트 로드 속도**           | 필요한 구성만 로드하기 때문에 상대적으로 빠름.                                                                  | 불필요한 구성 요소까지 로드할 가능성이 있어 상대적으로 느림.                                                    |
| **유연성**                      | 특정 요구사항에 맞춰 세부 설정을 직접 추가 및 제어 가능.                                                          | Spring Boot가 제공하는 기본 설정에 의존. 필요하면 일부 Bean만 Overriding 가능.                                     |
| **구현 편의성**                  | 구성 요소를 모두 직접 관리해야 하므로 상대적으로 복잡.                                                             | 대부분의 Kafka 관련 구성 요소를 Spring Boot가 자동으로 관리.                                                    |
| **필요한 프로퍼티 설정**          | Telemetry, Producer/Consumer 등 필요한 옵션만 최소화하여 설정 가능.                                                 | 모든 Kafka 관련 기본 프로퍼티를 사용. 추가 설정 필요 시 별도로 application.yml에 정의.                              |
| **테스트 격리**                  | `@DirtiesContext` 등을 통해 컨텍스트를 수동으로 정리하여 테스트 간 충돌 방지.                                          | Spring Boot가 제공하는 테스트 컨텍스트 격리 기능을 자동으로 사용.                                                 |
| **자동 추가되는 Bean**            | 없음. 모든 Bean을 직접 정의해야 함.                                                                               | KafkaAdmin, EmbeddedKafkaBroker, KafkaTemplate, ConsumerFactory 등 대부분의 Kafka 관련 Bean이 자동으로 등록. |

---

#### **5. `@AutoConfiguration`을 활용한 코드 예시**

아래는 `@AutoConfiguration` 기반으로 Kafka를 테스트하는 코드 예시입니다:

```java
@SpringBootTest
@EmbeddedKafka(partitions = 1, topics = {"test-topic"})
public class KafkaAutoConfigurationTest {

    @Autowired
    private KafkaTemplate<String, String> kafkaTemplate;

    @Test
    public void testKafkaMessagePublishAndConsume() throws InterruptedException {
        // 메시지 송신
        kafkaTemplate.send("test-topic", "key", "value");
        kafkaTemplate.flush();

        // 메시지 수신 확인
        Consumer<String, String> consumer = KafkaTestUtils.consumer("test-group", embeddedKafkaBroker);
        consumer.subscribe(List.of("test-topic"));
        ConsumerRecords<String, String> records = consumer.poll(Duration.ofSeconds(5));
        assertFalse(records.isEmpty(), "No records were found");
        ConsumerRecord<String, String> record = records.iterator().next();
        assertEquals("key", record.key());
        assertEquals("value", record.value());
        consumer.close();
    }
}
```

---

#### **6. 직접 구성 방식과 자동 구성 방식의 선택 기준**

- **직접 구성 방식이 적합한 경우**:
  - 루트 모듈과 격리된 테스트 환경을 구성해야 하는 경우.
  - Kafka 관련 최소 구성만 로드하여 컨텍스트 로드 속도를 최적화하려는 경우.
  - Spring Boot의 기본 자동 구성을 사용하지 못하는 환경.

- **자동 구성 방식이 적합한 경우**:
  - Spring Boot의 기본 설정을 최대한 활용하여 테스트 환경을 빠르게 구성하려는 경우.
  - Kafka 관련 모든 구성 요소를 자동으로 관리하여 구현 편의성을 높이고자 할 때.
  - Spring Kafka의 기본 동작이 요구사항에 충분히 부합하는 경우.

---

#### **7. 결론**

| **항목**             | **직접 구성 방식**                            | **자동 구성 방식**                           |
|--------------------|------------------------------------------|-----------------------------------------|
| **구성 복잡성**       | 세밀한 제어 가능, 복잡함                           | 단순하고 빠르게 설정 가능                         |
| **컨텍스트 로드 속도**  | 빠름                                       | 기본 Bean 로드로 인해 느릴 수 있음                |
| **유연성**           | 요구사항에 맞춰 세밀한 구성 가능                     | 기본 설정에 의존, 일부 Overriding 가능            |

상황에 따라 두 방식 중 적합한 방식을 선택하여 테스트 환경을 구성할 수 있습니다.  
**직접 구성 방식**은 최소한의 구성으로 성능 최적화와 테스트 격리가 중요한 경우 적합하며,  
**자동 구성 방식**은 구현의 편의성과 빠른 설정이 중요한 경우 더 적합합니다.
