이 둘의 차이점은 **Spring 애플리케이션 프로파일**의 활성화 방식과 `@ConditionalOnProperty`의 조건 평가 시점에 있습니다. 이를 하나씩 분석해 보겠습니다.

---

### 1. **첫 번째 케이스: `@ConditionalOnProperty`가 작동하지 않는 이유**

```java
@ContextConfiguration(classes = MailTestConfig.class)
@SpringBootTest
class MailServiceImplSpringBootTest {
}
```

#### 분석:
- **프로파일 활성화 누락**: 이 경우 `@Profile("test")`는 `MailTestConfig` 클래스에 지정되었지만, **어떤 프로파일이 활성화되었는지 명시**하지 않았습니다. 따라서 기본 프로파일(`default`)이 활성화됩니다.
- **`@ConditionalOnProperty` 조건 미충족**: `MailProperties.PREFIX.enabled` 속성이 `application.properties` 또는 `application-default.yml`에 설정되지 않았거나 `false`로 설정되어 있을 가능성이 큽니다.
  - 기본적으로 `@ConditionalOnProperty`는 지정된 프로퍼티가 존재하고, 값이 일치해야만 빈을 등록합니다.

결과적으로:
- `@Profile("test")` 조건이 충족되지 않음 → 클래스가 활성화되지 않음.
- `@ConditionalOnProperty` 조건도 충족되지 않음 → 빈 등록 실패.

---

### 2. **두 번째 케이스: `@Profile("test")`와 `@ActiveProfiles("test")`가 작동하는 이유**

```java
@ActiveProfiles("test") // application-test.yml 활성화
@ContextConfiguration(classes = MailTestConfig.class)
@SpringBootTest
class MailServiceImplSpringBootTest {
}
```

#### 분석:
- **`@ActiveProfiles("test")`로 프로파일 활성화**: `test` 프로파일이 활성화되었으므로, `@Profile("test")`가 적용된 `MailTestConfig` 클래스가 활성화됩니다.
- **`@ConditionalOnProperty` 조건 충족**: `test` 프로파일이 활성화되었으므로, `application-test.yml`이 로드됩니다. 이 파일에서 `MailProperties.PREFIX.enabled=true`로 설정되어 있다면 조건이 충족되어 빈이 등록됩니다.

결과적으로:
- `@Profile("test")`가 활성화 → `MailTestConfig` 클래스 로드.
- `application-test.yml`의 프로퍼티가 조건을 충족 → 빈 등록 성공.

---

### 3. **주요 차이점**

| **케이스**                         | **프로파일 활성화**            | **`@ConditionalOnProperty` 충족 여부** | **결과**                        |
|-----------------------------------|-------------------------------|----------------------------------------|---------------------------------|
| 첫 번째 (`@ActiveProfiles` 없음)   | 기본 프로파일 (`default`)      | 조건 미충족                             | 빈 등록 실패 (`MailTestConfig` 비활성화) |
| 두 번째 (`@ActiveProfiles("test")`) | `test` 프로파일 활성화         | 조건 충족                               | 빈 등록 성공 (`MailTestConfig` 활성화)  |

---

### 4. **작동 원리 요약**

1. **`@Profile("test")`**:
   - 클래스가 특정 프로파일에서만 활성화되도록 제한합니다.
   - 활성화되지 않은 프로파일에서는 해당 클래스가 스프링 컨텍스트에 포함되지 않습니다.

2. **`@ActiveProfiles("test")`**:
   - 테스트 실행 시 특정 프로파일을 활성화합니다.
   - `application-test.yml`을 로드하고, 해당 프로파일 관련 설정을 적용합니다.

3. **`@ConditionalOnProperty`**:
   - 설정된 프로퍼티 값에 따라 빈 등록 여부를 결정합니다.
   - 프로파일별 프로퍼티 파일(`application-test.yml`)의 값이 조건 충족 여부에 영향을 줍니다.

---

### 5. **정리**

- **첫 번째 케이스**: 기본 프로파일(`default`)이 활성화되며, `@Profile("test")` 조건을 충족하지 않아 설정 클래스가 로드되지 않고 빈 등록 실패.
- **두 번째 케이스**: `@ActiveProfiles("test")`로 `test` 프로파일을 활성화하면, `@Profile("test")`가 적용된 클래스가 로드되고, `@ConditionalOnProperty` 조건도 충족하여 빈 등록 성공.

이처럼 Spring Boot의 **프로파일 관리**와 **조건부 빈 등록**은 밀접하게 연관되어 있으며, 올바른 설정 파일과 활성 프로파일을 지정하는 것이 중요합니다. 😊
