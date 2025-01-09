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
  

### `@ActiveProfiles` 상세 설명

`@ActiveProfiles`는 Spring에서 테스트 실행 시 활성화할 **프로파일**을 지정하기 위해 사용되는 애너테이션입니다. 이 애너테이션은 주로 테스트 환경에서 사용되며, 프로파일 기반으로 서로 다른 설정을 로드하거나, 특정 테스트 설정을 적용할 수 있게 해줍니다.

---

### 1. `@ActiveProfiles`의 기본 동작

#### 정의
`@ActiveProfiles`는 Spring 테스트 컨텍스트에서 **활성화할 프로파일**을 지정하는 역할을 합니다. 프로파일은 `application-{profile}.properties` 또는 `application-{profile}.yml` 파일을 로드하거나, 특정 프로파일에만 적용되는 `@Configuration` 또는 빈(bean)을 활성화합니다.

```java
@ActiveProfiles("test")
@SpringBootTest
public class MyServiceTest {
    // 테스트 코드
}
```

#### 동작 방식
1. `@ActiveProfiles`에 지정된 프로파일이 Spring 컨텍스트 초기화 시 활성화됩니다.
2. 지정된 프로파일에 따라:
   - `application-{profile}.properties` 또는 `application-{profile}.yml` 파일이 로드됩니다.
   - `@Profile("{profile}")`가 적용된 클래스 또는 빈이 활성화됩니다.

---

### 2. `@ActiveProfiles`의 사용 사례

#### (1) 테스트 환경에서 `test` 프로파일 활성화
- 테스트 시 프로파일을 `test`로 지정하여 `application-test.yml` 설정을 로드합니다.
- 테스트 전용 빈과 설정이 활성화됩니다.

```java
@ActiveProfiles("test")
@SpringBootTest
public class UserServiceTest {
    @Autowired
    private UserService userService;

    @Test
    public void testUserCreation() {
        // 'test' 프로파일의 설정이 적용됨
        userService.createUser();
    }
}
```

#### (2) 다중 프로파일 활성화
- 여러 프로파일을 동시에 활성화할 수도 있습니다.

```java
@ActiveProfiles({"test", "mock"})
@SpringBootTest
public class UserServiceTest {
    // 'test'와 'mock' 프로파일 설정이 동시에 적용됨
}
```

---

### 3. 테스트 패키지와 메인 패키지에서의 차이점

#### (1) **테스트 패키지의 클래스에서 사용**
- `@ActiveProfiles`가 테스트 클래스에 적용되면 해당 테스트의 실행 시점에만 지정된 프로파일이 활성화됩니다.
- 이는 **테스트 컨텍스트**에서만 영향을 미치며, 실제 애플리케이션에는 영향을 주지 않습니다.
- 주로 테스트 환경에 적합한 설정을 적용하기 위해 사용됩니다.

```java
@ActiveProfiles("test")
@SpringBootTest
public class UserServiceTest {
    // 테스트 전용 프로파일 설정이 활성화됨
}
```

- **특징**:
  - `application-test.properties` 또는 `application-test.yml`이 로드됩니다.
  - 프로덕션 코드와 독립적인 테스트 환경을 구성할 수 있습니다.

---

#### (2) **메인 패키지의 클래스에서 사용**
- `@ActiveProfiles`가 메인 패키지의 클래스(비테스트 클래스)에 사용되면 애플리케이션 실행 시에도 적용될 수 있습니다.
- 이는 일반적으로 사용되지 않지만, 잘못된 위치에 있을 경우 **예상치 못한 프로파일 활성화** 문제를 일으킬 수 있습니다.

```java
@ActiveProfiles("prod")
public class MainConfig {
    // 일반적으로 테스트 환경 이외의 설정에는 사용하지 않음
}
```

- **특징**:
  - 프로파일 활성화는 `application.properties`에서 명시적으로 관리하는 것이 일반적입니다.
  - `@ActiveProfiles`를 메인 코드에 사용하면, 테스트와 실행 환경 간의 구분이 모호해질 수 있습니다.

---

### 4. `@ActiveProfiles` vs `application.properties` 설정

#### `application.properties`를 통한 프로파일 설정
- `spring.profiles.active` 속성을 통해 프로파일을 명시적으로 지정합니다.

```properties
spring.profiles.active=test
```

#### `@ActiveProfiles`와의 차이
- `application.properties`는 전체 애플리케이션 실행에 영향을 미칩니다.
- `@ActiveProfiles`는 테스트 클래스의 **특정 컨텍스트**에서만 프로파일을 활성화합니다.

---

### 5. 주의 사항 및 활용 팁

1. **테스트용으로만 사용**:
   - `@ActiveProfiles`는 주로 테스트 클래스에서 사용하며, 메인 패키지의 클래스에는 사용하지 않는 것이 좋습니다.

2. **프로파일 분리 전략**:
   - 프로덕션과 테스트 환경의 설정을 명확히 분리하고, `@ActiveProfiles`로 테스트 환경을 활성화합니다.

3. **다중 프로파일 충돌 주의**:
   - 다중 프로파일을 지정할 때 설정 간 충돌이 발생할 수 있으므로, 필요한 프로파일만 활성화하도록 제한합니다.

4. **속성 우선순위**:
   - `@ActiveProfiles`로 활성화된 프로파일은 `spring.profiles.active` 설정을 덮어씁니다.

---

### 6. 정리

| **특징**                          | **테스트 패키지에서 사용**              | **메인 패키지에서 사용**               |
|----------------------------------|-------------------------------------|------------------------------------|
| **영향 범위**                    | 테스트 클래스의 컨텍스트에만 적용        | 전체 애플리케이션 실행에 영향을 미칠 수 있음 |
| **사용 목적**                    | 테스트 환경에 맞는 설정 적용             | (비추천) 프로파일 관리 오류 발생 가능       |
| **추천 사용 위치**                | 테스트 클래스                          | 테스트 환경 설정만 관리               |
| **기본 설정과의 관계**             | `spring.profiles.active`를 덮어씀       | 설정 충돌 발생 가능                  |

이처럼 `@ActiveProfiles`는 테스트 환경을 설정하는 강력한 도구이며, 적절한 사용 위치와 목적을 이해하면 효율적인 테스트 환경을 구성할 수 있습니다. 😊

---

### 5. **정리**

- **첫 번째 케이스**: 기본 프로파일(`default`)이 활성화되며, `@Profile("test")` 조건을 충족하지 않아 설정 클래스가 로드되지 않고 빈 등록 실패.
- **두 번째 케이스**: `@ActiveProfiles("test")`로 `test` 프로파일을 활성화하면, `@Profile("test")`가 적용된 클래스가 로드되고, `@ConditionalOnProperty` 조건도 충족하여 빈 등록 성공.

이처럼 Spring Boot의 **프로파일 관리**와 **조건부 빈 등록**은 밀접하게 연관되어 있으며, 올바른 설정 파일과 활성 프로파일을 지정하는 것이 중요합니다. 😊
