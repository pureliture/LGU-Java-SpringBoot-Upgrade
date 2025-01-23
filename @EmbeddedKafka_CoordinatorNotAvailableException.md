### **Kafka Embedded 환경에서 발생하는 CoordinatorNotAvailableException: 원인과 해결이 필요 없는 이유**

---

#### **1. 배경**

테스트 환경에서 **Spring Kafka**의 **Embedded Kafka**를 사용하며 메시지 송신(Publish) 및 수신(Subscribe)을 검증하던 중, 다음과 같은 **`CoordinatorNotAvailableException`**이 발생하는 로그를 관찰할 수 있었습니다:

```
org.apache.kafka.common.errors.CoordinatorNotAvailableException: The coordinator is not available.
```

이 오류는 특히 **Consumer 시작 시점**에서 발생하며, 이후에는 정상적으로 메시지 송신 및 수신이 이루어집니다. 본 포스트에서는 이 오류가 발생하는 **원인**과 **문제가 되지 않는 이유**를 Kafka의 동작 원리를 바탕으로 설명합니다.

---

#### **2. CoordinatorNotAvailableException이란?**

Kafka의 **Consumer Group Coordinator**는 다음과 같은 역할을 합니다:
- Consumer 그룹의 멤버십 관리.
- 파티션 할당 및 오프셋 커밋 관리.

**Consumer Group Coordinator**를 찾기 위해, Kafka Consumer는 브로커에 **`FindCoordinator`** 요청을 전송합니다. 하지만 다음 상황에서 **`CoordinatorNotAvailableException`**이 발생할 수 있습니다:
1. 그룹 코디네이터가 아직 등록되지 않았거나, 활성화되지 않은 상태.
2. 브로커 초기화와 Consumer의 요청 타이밍이 맞지 않는 경우.

---

#### **3. 오류 발생 원인**

테스트 환경에서 이 오류는 주로 다음 원인으로 인해 발생합니다:

##### **3.1. Embedded Kafka 초기화 타이밍**
Embedded Kafka는 실제 Kafka 클러스터와 다르게 경량화된 구조로 동작하며, 브로커 초기화와 메타데이터(예: 그룹 코디네이터) 등록에 시간이 소요됩니다. 이때 Consumer가 너무 빨리 그룹 코디네이터를 찾으려 하면 실패하게 됩니다.

- **Consumer 요청 → Kafka 응답 시점의 문제**  
  요청이 너무 빨리 전송된 경우, 응답에서 **`nodeId=-1`**, **`host=''`**, **`port=-1`** 등이 반환되며 Coordinator를 찾지 못합니다.

##### **3.2. FindCoordinator 요청의 기본 동작**
Kafka는 그룹 코디네이터를 찾기 위해 **`FindCoordinator`** 요청을 반복적으로 전송하며, 첫 번째 요청이 실패해도 이후 메타데이터를 갱신하며 다시 시도합니다. 이 동작은 Kafka의 기본적인 **Coordinator Discovery** 과정에서 자연스럽게 발생할 수 있습니다.

##### **3.3. Auto Topic Creation 비활성화**
테스트에서 구독하려는 토픽이 존재하지 않을 경우, 브로커가 코디네이터를 정상적으로 할당하지 못할 가능성이 있습니다. 이로 인해 첫 번째 요청이 실패할 수 있습니다.

---

#### **4. 문제가 없는 이유**

**CoordinatorNotAvailableException**이 테스트에서 문제가 되지 않는 이유는 다음과 같습니다:

##### **4.1. Kafka의 기본적인 동작**
Kafka는 FindCoordinator 요청이 실패한 경우에도 메타데이터 갱신을 통해 재시도하며, Coordinator를 정상적으로 찾을 때까지 반복합니다. 따라서:
- 첫 번째 요청이 실패해도 Consumer는 이후 재시도에서 성공적으로 그룹 코디네이터를 찾습니다.
- 실제 메시지 송신 및 수신에는 영향을 미치지 않습니다.

##### **4.2. 로그는 초기화 과정의 일부**
이 오류는 주로 테스트 시작 시점에서 발생하며, 이후 정상 동작합니다. 이는 Kafka Embedded 환경에서 초기화 과정 중 메타데이터 등록이 완료되기 전에 발생하는 일시적인 현상으로 간주됩니다.

##### **4.3. 테스트 실패 없음**
이 오류가 발생했음에도 불구하고, 테스트는 정상적으로 동작하며 메시지가 올바르게 송신 및 수신됩니다. 따라서 테스트 결과에 영향을 미치지 않습니다.

---

#### **5. 해결 방법이 필요하지 않은 이유**

이 문제는 Kafka의 정상적인 동작 과정 중 일부이며, 테스트 환경에서 무시해도 되는 로그입니다. 그러나 로그를 깨끗하게 유지하거나 오류 발생을 방지하고 싶다면 다음 조치를 고려할 수 있습니다:

1. **Consumer 시작 전 대기**  
   Embedded Kafka가 완전히 초기화될 때까지 대기합니다.
   ```java
   @BeforeEach
   void setup() throws InterruptedException {
       Thread.sleep(1000); // 1초 대기
   }
   ```

2. **메타데이터 갱신 간격 조정**  
   FindCoordinator 실패 시 더 빠르게 재시도하도록 간격을 줄입니다.
   ```yaml
   spring:
     kafka:
       consumer:
         properties:
           retry.backoff.ms: 200 # 재시도 간격
           metadata.max.age.ms: 1000 # 메타데이터 갱신 주기
   ```

3. **Auto Topic Creation 활성화**  
   테스트에서 사용되는 토픽이 사전에 생성되지 않은 경우 자동으로 생성되도록 활성화합니다.
   ```java
   @EmbeddedKafka(partitions = 1, topics = {"test-topic"}, brokerProperties = {"auto.create.topics.enable=true"})
   ```

---

#### **6. 결론**

CoordinatorNotAvailableException은 Kafka의 초기화 및 Coordinator Discovery 과정에서 발생할 수 있는 **정상적인 현상**입니다. 특히 테스트 환경에서 Embedded Kafka를 사용할 경우 발생 가능성이 높으며, 이를 해결하지 않아도 다음 조건을 만족한다면 문제가 없습니다:
1. Consumer가 재시도를 통해 정상적으로 그룹 코디네이터를 찾는다.
2. 메시지 송신 및 수신이 정상적으로 이루어진다.
3. 테스트 결과에 영향을 미치지 않는다.

만약 로그를 줄이고 초기화 과정을 최적화하고 싶다면 위에서 제시한 방법을 적용할 수 있습니다. 하지만 테스트 동작에 영향이 없다면 굳이 수정하지 않고 로그를 무시해도 무방합니다.
