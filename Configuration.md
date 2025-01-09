### `@AutoConfiguration` vs `@Configuration`

Spring Boot에서는 어플리케이션의 빈(bean) 설정을 정의하기 위해 `@Configuration`을 사용하며, 자동 구성(auto-configuration)을 제공하기 위해 `@AutoConfiguration`이 추가되었습니다. 두 애너테이션의 주요 차이점과 용도는 다음과 같습니다.

---

### 1. `@Configuration`
`@Configuration`은 Spring에서 빈을 명시적으로 정의하기 위한 애너테이션입니다. 이 애너테이션이 붙은 클래스는 하나 이상의 `@Bean` 메서드를 포함하며, 해당 메서드들이 반환하는 객체를 스프링 컨텍스트에 빈으로 등록합니다.

```java
@Configuration
public class AppConfig {

    @Bean
    public MyService myService() {
        return new MyService();
    }

    @Bean
    public MyRepository myRepository() {
        return new MyRepository();
    }
}
```

**특징:**
- 명시적으로 빈을 정의합니다.
- 일반적으로 개발자가 직접 작성한 설정 클래스에서 사용됩니다.
- Spring의 **클래스 기반 설정**을 활성화합니다.
- Spring Boot의 자동 구성(`@AutoConfiguration`)과 달리 모든 설정이 수동으로 관리됩니다.

---

### 2. `@AutoConfiguration`
`@AutoConfiguration`은 Spring Boot 3.0에서 도입된 애너테이션으로, 자동 구성을 위한 설정 클래스를 정의하는 데 사용됩니다. 자동 구성을 통해 Spring Boot는 애플리케이션에서 자주 사용되는 설정을 미리 제공하며, 이 설정은 클래스패스와 환경에 따라 자동으로 활성화됩니다.

```java
@AutoConfiguration
public class MyAutoConfiguration {

    @Bean
    public MyService myService() {
        return new MyService();
    }
}
```

**특징:**
- Spring Boot에서 제공하는 **자동 구성 메커니즘**을 확장하기 위해 사용됩니다.
- `spring.factories` 또는 `spring.autoconfigure` 파일에 등록하여 조건부로 활성화됩니다.
- **클래스패스와 환경 설정**에 따라 동적으로 구성됩니다.
- 주로 라이브러리 또는 공통 설정을 자동화하는 데 사용됩니다.

---

### 주요 차이점

| **특징**              | `@Configuration`                         | `@AutoConfiguration`                        |
|-----------------------|------------------------------------------|---------------------------------------------|
| **사용 목적**         | 명시적인 빈 정의                          | 자동 구성 클래스 정의                         |
| **사용 시점**         | 애플리케이션 또는 특정 모듈의 커스텀 설정  | Spring Boot 자동 구성 확장을 위해 사용        |
| **조건부 활성화**     | 없음                                     | 클래스패스 및 환경에 따라 활성화              |
| **등록 방법**         | 직접 Spring 컨텍스트에서 사용             | `spring.factories` 또는 `spring.autoconfigure` 등록 |

---

### 3. `@Configuration`에 `@Profile("test")` 추가 시 동작 변화

`@Profile` 애너테이션은 특정 프로파일이 활성화된 경우에만 해당 설정 클래스나 빈이 로드되도록 제한합니다. `@Configuration` 클래스에 `@Profile`을 추가하면, 해당 프로파일이 활성화되지 않는 한 클래스 내부의 모든 설정은 무시됩니다.

#### 예제

```java
@Configuration
@Profile("test")
public class TestConfig {

    @Bean
    public TestService testService() {
        return new TestService();
    }
}
```

**동작:**
- `spring.profiles.active=test`로 설정된 경우에만 `TestConfig` 클래스가 로드됩니다.
- 다른 프로파일이 활성화되면 `TestConfig` 클래스와 그 내부의 빈은 스프링 컨텍스트에 포함되지 않습니다.

#### 프로파일 활성화 방법
- **application.properties**
  ```properties
  spring.profiles.active=test
  ```
- **JVM 옵션**
  ```
  -Dspring.profiles.active=test
  ```
- **코드 내 활성화**
  ```java
  ConfigurableApplicationContext context = 
      new SpringApplicationBuilder(App.class)
          .profiles("test")
          .run(args);
  ```

#### 장점
1. **환경별 구분된 설정 관리**: 개발, 테스트, 프로덕션 환경에 따라 다른 설정을 로드할 수 있습니다.
2. **클래스 수준 관리**: 프로파일 조건을 클래스 전체에 적용할 수 있습니다.

---

### 결론
1. `@Configuration`은 명시적인 설정을 위한 클래스로 사용하며, 개발자가 직접 빈 정의를 관리합니다.
2. `@AutoConfiguration`은 Spring Boot의 자동 구성을 확장하는 데 사용되며, 환경과 조건에 따라 동적으로 빈을 구성합니다.
3. `@Profile`은 환경별로 특정 설정을 로드하거나 제외하는 데 유용하며, 이를 통해 테스트와 프로덕션 환경의 설정을 명확히 분리할 수 있습니다.

이러한 특징을 기반으로 설정 관리 전략을 수립하면 효율적인 Spring Boot 애플리케이션 개발이 가능합니다. 😊
