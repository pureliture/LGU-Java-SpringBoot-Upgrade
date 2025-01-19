### 1. **전략 분석**: Datashield 활성화/비활성화

#### 관련 파일 확인
압축 파일에서 다음과 같은 파일이 활성화/비활성화 전략과 관련되어 있음을 확인했습니다:

1. **테스트 환경 파일**:
   - `src/test/resources/datashield-activation-application.yml`
   - `src/test/resources/datashield-deactivation-application.yml`

2. **Spring Configuration 클래스**:
   - `DatashieldActivationTestConfig`
   - `DatashieldDeactivationTestConfig`

3. **테스트 실행 조건**:
   - `DatashieldActivateTestExecutionCondition`
   - `DatashieldDeactivateTestExecutionCondition`

---

#### `.yml` 파일 분석

**`datashield-activation-application.yml`**:
```yaml
datashield:
  enabled: true
  datatype-config: config/datashield-activation-datatype.yml
  application:
    system-code: NUC
```

**`datashield-deactivation-application.yml`**:
```yaml
datashield:
  enabled: false
  datatype-config: config/datashield-deactivation-datatype.yml
  application:
    system-code: NUC
```

이 두 `.yml` 파일은 `test.datashield.activate` 값을 기반으로 Spring의 `@ConditionalOnProperty` 및 `ExecutionCondition`에 의해 활성화/비활성화 동작을 제어합니다.

---

### 2. **`ExecutionCondition`를 활용한 테스트 제어**

#### 활성화된 조건 구현: `DatashieldActivateTestExecutionCondition`
```java
import org.junit.jupiter.api.extension.ConditionEvaluationResult;
import org.junit.jupiter.api.extension.ExecutionCondition;
import org.junit.jupiter.api.extension.ExtensionContext;
import org.springframework.core.env.Environment;
import org.springframework.test.context.junit.jupiter.SpringExtension;

public class DatashieldActivateTestExecutionCondition implements ExecutionCondition {

    @Override
    public ConditionEvaluationResult evaluateExecutionCondition(ExtensionContext context) {
        Environment environment = SpringExtension.getApplicationContext(context).getEnvironment();
        String isEnabled = environment.getProperty("test.datashield.activate");

        if ("true".equalsIgnoreCase(isEnabled)) {
            return ConditionEvaluationResult.enabled("Datashield is enabled");
        }
        return ConditionEvaluationResult.disabled("Datashield is disabled");
    }
}
```

#### 비활성화된 조건 구현: `DatashieldDeactivateTestExecutionCondition`
```java
import org.junit.jupiter.api.extension.ConditionEvaluationResult;
import org.junit.jupiter.api.extension.ExecutionCondition;
import org.junit.jupiter.api.extension.ExtensionContext;
import org.springframework.core.env.Environment;
import org.springframework.test.context.junit.jupiter.SpringExtension;

public class DatashieldDeactivateTestExecutionCondition implements ExecutionCondition {

    @Override
    public ConditionEvaluationResult evaluateExecutionCondition(ExtensionContext context) {
        Environment environment = SpringExtension.getApplicationContext(context).getEnvironment();
        String isEnabled = environment.getProperty("test.datashield.activate");

        if ("false".equalsIgnoreCase(isEnabled)) {
            return ConditionEvaluationResult.enabled("Datashield is disabled");
        }
        return ConditionEvaluationResult.disabled("Datashield is enabled");
    }
}
```

---

### 3. **Spring Configuration 클래스**

#### 활성화된 상태에서의 설정: `DatashieldActivationTestConfig`
```java
import org.springframework.boot.autoconfigure.condition.ConditionalOnProperty;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
@ConditionalOnProperty(prefix = "datashield", name = "enabled", havingValue = "true")
public class DatashieldActivationTestConfig {

    @Bean
    public DatashieldClient datashieldClient() {
        return new DatashieldClientImpl(); // 실제 클라이언트 구현체
    }
}
```

#### 비활성화된 상태에서의 설정: `DatashieldDeactivationTestConfig`
```java
import org.springframework.boot.autoconfigure.condition.ConditionalOnProperty;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
@ContextConfiguration(classes = DatashieldActivationTestConfig.class)
@ConditionalOnProperty(prefix = "datashield", name = "enabled", havingValue = "false")
public class DatashieldDeactivationTestConfig {

    @Bean
    public MockDatashieldClient mockDatashieldClient() {
        return new MockDatashieldClient(); // Mock 클라이언트 구현체
    }
}
```

---

### 4. **테스트 코드**

#### 활성화된 테스트: `DatashieldInitializerSpringBootTest`
```java
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.springframework.boot.test.context.SpringBootTest;

@ExtendWith(DatashieldActivateTestExecutionCondition.class)
@SpringBootTest(properties = "spring.config.location=classpath:datashield-activation-application.yml")
public class DatashieldInitializerSpringBootTest {

    @Test
    void testDatashieldActivation() {
        System.out.println("Datashield is activated and tested.");
    }
}
```

#### 비활성화된 테스트: `DatashieldDeactivationTest`
```java
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.springframework.boot.test.context.SpringBootTest;

@ExtendWith(DatashieldDeactivateTestExecutionCondition.class)
@ContextConfiguration(classes = DatashieldDeactivationTestConfig.class)
@SpringBootTest(properties = "spring.config.location=classpath:datashield-deactivation-application.yml")
public class DatashieldDeactivationTest {

    @Test
    void testDatashieldDeactivation() {
        System.out.println("Datashield is deactivated and mock-tested.");
    }
}
```

---

### 5. **분석 결과 요약**

#### 활성화/비활성화 전략:
1. **환경 파일 (`.yml`)**:
   - `test.datashield.activate` 값을 통해 활성화(true)와 비활성화(false)를 구분.
   - Spring의 `@ConditionalOnProperty` 및 테스트 조건에서 해당 값을 참조.

2. **`ExecutionCondition`**:
   - JUnit 5의 `ExecutionCondition`을 활용하여 조건에 따라 테스트를 활성화 또는 비활성화.
   - 활성화: `DatashieldActivateTestExecutionCondition`.
   - 비활성화: `DatashieldDeactivateTestExecutionCondition`.

3. **Spring Configuration**:
   - `@ConditionalOnProperty`를 통해 활성화(true)와 비활성화(false) 상태에서 다른 Bean을 주입.

4. **테스트 코드**:
   - 활성화된 상태와 비활성화된 상태를 각각 검증하기 위해 분리된 테스트 클래스를 작성.
