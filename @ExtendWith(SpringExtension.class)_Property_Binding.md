`@ExtendWith(SpringExtension.class)`를 사용하는 경우, **`PropertySourcesPlaceholderConfigurer`가 필수는 아닙니다.**

다만, **Spring Framework의 기본 컨텍스트에서 외부 프로퍼티 파일을 사용**하려고 한다면, `PropertySourcesPlaceholderConfigurer` 또는 그에 상응하는 설정이 필요할 수 있습니다. 이는 Spring Boot와의 차이에서 기인하며, 다음과 같은 원칙에 따라 달라집니다.

---

### **1. Spring Framework vs Spring Boot**

#### **Spring Framework (순수 Framework 프로젝트)**

Spring Framework에서는 `PropertySourcesPlaceholderConfigurer`를 명시적으로 정의해야 외부 프로퍼티 파일을 로드하고, `${}` 형식의 플레이스홀더를 처리할 수 있습니다.

##### **예시: Spring Framework에서 필요**
```java
@ExtendWith(SpringExtension.class)
@ContextConfiguration(classes = AppConfig.class)
class MyTest {

    @TestConfiguration
    static class AppConfig {
        @Bean
        public static PropertySourcesPlaceholderConfigurer propertyConfigurer() {
            return new PropertySourcesPlaceholderConfigurer();
        }

        @Bean
        public MyBean myBean(@Value("${my.property}") String myProperty) {
            return new MyBean(myProperty);
        }
    }

    @Test
    void testPropertyInjection() {
        // Test logic
    }
}
```

#### **Spring Boot (부트 기반 프로젝트)**

Spring Boot는 자동으로 외부 프로퍼티 파일을 로드하고, 플레이스홀더를 처리합니다. Spring Boot 환경에서는 `PropertySourcesPlaceholderConfigurer`를 명시적으로 정의하지 않아도 됩니다.

```java
@ExtendWith(SpringExtension.class)
@SpringBootTest
class MyTest {
    @Value("${my.property}")
    private String myProperty;

    @Test
    void testPropertyInjection() {
        assertEquals("expectedValue", myProperty);
    }
}
```

**Spring Boot가 자동으로 처리하는 이유**:
1. `@SpringBootTest`는 `application.properties` 또는 `application.yml` 파일을 자동으로 로드합니다.
2. Spring Boot는 내부적으로 `PropertySourcesPlaceholderConfigurer`를 자동 등록합니다.

---

### **2. 언제 `PropertySourcesPlaceholderConfigurer`가 필요한가?**

`@ExtendWith(SpringExtension.class)`를 사용하는 환경에서 다음과 같은 상황이라면 `PropertySourcesPlaceholderConfigurer`가 필요합니다.

#### **a. Spring Boot를 사용하지 않는 경우**
Spring Boot가 아닌 순수 Spring Framework 환경에서 외부 프로퍼티를 사용하려면, `PropertySourcesPlaceholderConfigurer`를 명시적으로 정의해야 합니다.

#### **b. Bean 정의에 `${}` 형식의 플레이스홀더를 사용하는 경우**
`@Value` 또는 XML 설정에서 `${}` 플레이스홀더를 사용하려면 `PropertySourcesPlaceholderConfigurer`를 정의해야 합니다.

#### **c. 프로퍼티 파일을 명시적으로 로드해야 하는 경우**
특정 경로의 프로퍼티 파일을 직접 로드하려는 경우에도 필요합니다.

```java
@PropertySource("classpath:custom.properties")
public class AppConfig {
    @Bean
    public static PropertySourcesPlaceholderConfigurer propertyConfigurer() {
        return new PropertySourcesPlaceholderConfigurer();
    }
}
```

---

### **3. 언제 `PropertySourcesPlaceholderConfigurer`가 필요하지 않은가?**

#### **a. Spring Boot 사용 시**
Spring Boot는 외부 프로퍼티 파일 처리와 플레이스홀더 치환을 자동으로 처리하므로 명시적으로 설정할 필요가 없습니다.

#### **b. 외부 프로퍼티 파일을 사용하지 않는 경우**
플레이스홀더 치환이나 외부 프로퍼티 파일이 필요하지 않다면 정의할 필요가 없습니다.

#### **c. `@SpringBootTest` 사용 시**
`@SpringBootTest`는 컨텍스트 내에서 모든 환경 프로퍼티를 자동으로 로드하므로, 추가 설정 없이 외부 프로퍼티 파일을 사용할 수 있습니다.

---

### **4. 사용 여부 결정 기준**

| **조건**                                | **필요 여부**        |
|----------------------------------------|---------------------|
| Spring Boot 사용                       | 필요하지 않음        |
| 순수 Spring Framework 사용             | 필요                |
| 외부 프로퍼티 파일 사용                | 필요                |
| `@Value("${}")` 사용                   | 필요                |
| 자동 컨텍스트 로드 및 플레이스홀더 필요 | Spring Boot: 필요 없음 |

---

### **5. 요약**
- `@ExtendWith(SpringExtension.class)`를 사용할 때 **Spring Boot를 사용**한다면 `PropertySourcesPlaceholderConfigurer`는 필요하지 않습니다. Spring Boot가 이를 자동으로 처리합니다.
- 순수 **Spring Framework 환경에서 외부 프로퍼티 파일을 사용**하려면, `PropertySourcesPlaceholderConfigurer`를 명시적으로 정의해야 합니다.

추가적으로 도움이 필요하면 언제든지 말씀해주세요! 😊
