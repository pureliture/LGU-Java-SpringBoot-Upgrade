### **Spring Kafka 테스트 환경 구성: 필수 설정만으로 모듈 검증하기**

---

#### **1. 프로젝트 배경**

테스트 중인 모듈은 대규모 프레임워크의 서브 모듈로, 루트 모듈(Web, REST, MVC 등)을 로드하지 않고 **Spring Kafka**를 활용해 독립적으로 메시지 송신 및 수신을 검증하려는 목적으로 구성되었습니다.  
이 과정에서 **`@SpringBootApplication`**과 같은 루트 레벨 자동 구성을 사용할 수 없으며, 필요한 구성 요소만 명시적으로 추가해야 했습니다.

---

#### **2. 주요 문제점**

1. **`@SpringBootApplication` 사용 불가**  
   루트 모듈이 포함된 자동 구성을 피하기 위해 필요한 구성 요소를 직접 추가해야 했습니다.

2. **Telemetry 및 Metric Push 문제**  
   Kafka 클라이언트(3.7.1)에서 Telemetry 및 Metric Push 기능으로 인해 **UnsupportedVersionException**이 발생할 수 있었습니다.

3. **Embedded Kafka 리소스 오염**  
   테스트 실행 시 **Embedded Kafka** 리소스가 이전 테스트와 충돌하거나 잔여 데이터로 인해 결과가 달라질 가능성이 있었습니다.

4. **EOFException**  
   Kafka 클라이언트 종료 시 **Graceful Shutdown** 과정에서 발생하는 **EOFException**이 문제가 되는 듯 보였으나, 이는 실제로 무시 가능한 예외로 확인되었습니다.

---

#### **3. 테스트 환경 구성**

##### **3.1. `@ContextConfiguration`으로 필요한 설정만 추가**
`KafkaAutoConfiguration`을 명시적으로 추가하여 Kafka 관련 빈만 로드하도록 설정했습니다.

```java
@ContextConfiguration(classes = {KafkaAutoConfiguration.class)
@SpringBootTest
@EmbeddedKafka(partitions = 1, topics = {"test-topic"})
@DirtiesContext(classMode = DirtiesContext.ClassMode.AFTER_CLASS)
public class KafkaStreamInspectorAspectorSpringBootTest { ... }
```

- **`KafkaAutoConfiguration`**: Kafka 관련 Bean 생성 및 설정 로드.
- **`@EmbeddedKafka`**: 테스트 중 Kafka 브로커와 토픽 설정을 제공.
- **`@DirtiesContext`**: Embedded Kafka 사용 시 컨텍스트를 새로 생성하여 리소스 오염 방지.

---

##### **3.2. 필수 환경 변수**

`application.yml`에서 테스트에 필요한 최소 설정만 유지했습니다.

```yaml
spring:
  kafka:
    bootstrap-servers: ${spring.embedded.kafka.brokers} # Embedded Kafka 브로커 주소
    producer:
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.apache.kafka.common.serialization.StringSerializer
    consumer:
      group-id: test-group # Consumer 그룹 ID
      key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      value-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      auto-offset-reset: earliest # 메시지를 처음부터 소비
      enable-auto-commit: false # 오프셋 자동 커밋 비활성화
    properties:
      telemetry.enable: false # Telemetry 비활성화
      enable.metrics.push: false # Metric Push 비활성화
      telemetry.endpoint: "" # Telemetry 엔드포인트 비활성화
```

---

#### **4. 각 설정의 역할**

| **설정 키**                          | **설명**                                                                                               |
|--------------------------------------|-------------------------------------------------------------------------------------------------------|
| **`bootstrap-servers`**              | Embedded Kafka 브로커 주소를 동적으로 참조. 테스트 실행 중 필요한 브로커 연결 정보를 제공합니다.                                              |
| **`producer.key-serializer`**        | 메시지 키를 문자열로 직렬화. Kafka 메시지 전송 시 필수.                                                          |
| **`producer.value-serializer`**      | 메시지 값을 문자열로 직렬화. Kafka 메시지 전송 시 필수.                                                          |
| **`consumer.group-id`**              | Consumer 그룹 ID 설정. 그룹 내 메시지 분산을 위해 필수.                                                          |
| **`consumer.key-deserializer`**      | 메시지 키를 문자열로 역직렬화. Kafka 메시지 소비 시 필수.                                                        |
| **`consumer.value-deserializer`**    | 메시지 값을 문자열로 역직렬화. Kafka 메시지 소비 시 필수.                                                        |
| **`consumer.auto-offset-reset`**     | Consumer가 오프셋을 찾을 수 없을 때 처음부터 소비하도록 설정. 메시지 누락 방지를 위해 필수.                                              |
| **`consumer.enable-auto-commit`**    | Consumer의 자동 커밋 비활성화. 메시지 처리를 수동으로 제어하려면 필수.                                              |
| **`properties.telemetry.enable`**    | Telemetry 비활성화. Kafka 클라이언트의 불필요한 네트워크 호출 및 오류를 방지하기 위해 필요.                                   |
| **`properties.enable.metrics.push`** | Metric Push 비활성화. Telemetry와 유사하게 불필요한 네트워크 호출 방지를 위해 필요.                                   |
| **`properties.telemetry.endpoint`**  | Telemetry 엔드포인트를 비활성화. Telemetry와 관련된 모든 네트워크 요청을 차단.                                            |

---

#### **5. 테스트 코드**

아래는 Kafka 메시지 발행(Publish) 및 구독(Subscribe)을 검증하는 테스트 코드입니다.

```java
@Test
void testFetchPubMsgTopicList() throws InterruptedException, ExecutionException {
    // Consumer 생성 및 Topic 구독
    Consumer<String, String> consumer = consumerFactory.createConsumer();
    consumer.subscribe(List.of("test-topic"));

    // 메시지 발행
    kafkaTemplate.send("test-topic", "test-key", "test-message").get();
    kafkaTemplate.flush();

    // 메시지 구독 확인
    Thread.sleep(1000);
    ConsumerRecords<String, String> records = consumer.poll(Duration.ofSeconds(5));
    assertFalse(records.isEmpty(), "No records were found");

    ConsumerRecord<String, String> record = records.iterator().next();
    assertEquals("test-key", record.key());
    assertEquals("test-message", record.value());

    // Consumer 종료
    consumer.close();
}
```

---

#### **6. 발생 가능 문제 및 해결**

##### **6.1. Telemetry 및 Metric Push 문제**
- **문제**: Kafka 클라이언트 3.7.1에서 Telemetry 및 Metric Push 기능 활성화로 인해 **UnsupportedVersionException** 발생.
- **해결**: 아래 설정으로 비활성화.
  ```yaml
  spring:
    kafka:
      properties:
        telemetry.enable: false
        enable.metrics.push: false
        telemetry.endpoint: ""
  ```

##### **6.2. EOFException**
- **문제**: Graceful Shutdown 중 **EOFException** 발생.
- **해결**: 테스트 동작에는 영향을 주지 않으므로 무시. 추가적인 클라이언트 종료 제어가 필요하지 않음.

##### **6.3. Embedded Kafka 리소스 충돌**
- **문제**: 테스트 간 리소스 충돌로 인해 테스트 결과가 일관되지 않을 가능성.
- **해결**: `@DirtiesContext`를 통해 컨텍스트를 새로 생성.
  ```java
  @DirtiesContext(classMode = DirtiesContext.ClassMode.AFTER_CLASS)
  ```

---

#### **7. 최종 정리**

| **항목**                          | **설정/해결 방법**                                                                                           |
|----------------------------------|----------------------------------------------------------------------------------------------------------|
| **KafkaAutoConfiguration 사용**   | `@ContextConfiguration(classes = {KafkaAutoConfiguration.class})`로 명시적으로 추가.                      |
| **Embedded Kafka 설정**           | `@EmbeddedKafka(partitions = 1, topics = {"test-topic"})`.                                               |
| **필수 환경 변수 설정**           | Telemetry 및 Metric Push 비활성화, Consumer와 Producer 설정.                                               |
| **컨텍스트 정리**                 | `@DirtiesContext(classMode = DirtiesContext.ClassMode.AFTER_CLASS)`.                                     |
| **EOFException 무시**             | 테스트 동작에는 영향이 없으므로 추가 제어 없이 그대로 무시.                                                   |
