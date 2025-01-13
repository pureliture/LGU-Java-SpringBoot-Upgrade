`DisposableBean`은 Spring 프레임워크에서 제공하는 인터페이스로, Spring Bean이 소멸될 때 특정 작업을 실행할 수 있도록 지원합니다. Spring Boot 3.3.5 (Spring Framework 6.x 기반)에서도 이 인터페이스를 사용할 수 있습니다.

### DisposableBean의 개념
- **목적:** Bean의 생명 주기에서 **소멸 단계**에 실행할 로직을 정의합니다.
- **사용 대상:** 주로 외부 리소스를 정리하거나 연결을 종료해야 할 때 사용됩니다. 예를 들면:
  - 데이터베이스 연결 종료
  - 파일 핸들러 닫기
  - 메시지 브로커 연결 해제

### 주요 메서드
`DisposableBean` 인터페이스는 단 하나의 메서드를 제공합니다:

```java
void destroy() throws Exception;
```

이 메서드는 Bean이 컨테이너에서 제거될 때 호출됩니다.

---

### 사용 방법
Spring에서 `DisposableBean`을 구현하여 사용하려면, Bean 클래스에서 `destroy()` 메서드를 구현해야 합니다.

#### 예제 코드
```java
import org.springframework.beans.factory.DisposableBean;
import org.springframework.stereotype.Component;

@Component
public class MyDisposableBean implements DisposableBean {

    @Override
    public void destroy() throws Exception {
        System.out.println("Bean is being destroyed. Cleaning up resources...");
        // 여기에 리소스 정리 로직 작성
    }
}
```

위의 코드에서:
1. `MyDisposableBean` 클래스는 `DisposableBean`을 구현.
2. Spring 컨테이너에서 이 Bean이 제거될 때 `destroy()` 메서드가 호출됩니다.

---

### 다른 리소스 정리 방법과 비교
Spring 3.x부터는 `@PreDestroy` 어노테이션 또는 `@Bean(destroyMethod)`를 사용하는 방식이 더 선호됩니다. 이는 특정 인터페이스에 의존하지 않기 때문에 더 유연합니다.

#### @PreDestroy 방식
```java
import jakarta.annotation.PreDestroy;
import org.springframework.stereotype.Component;

@Component
public class MyBean {

    @PreDestroy
    public void cleanup() {
        System.out.println("Bean is being destroyed using @PreDestroy.");
    }
}
```

#### destroyMethod 방식
```java
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class AppConfig {

    @Bean(destroyMethod = "cleanup")
    public MyCustomBean myCustomBean() {
        return new MyCustomBean();
    }
}

class MyCustomBean {

    public void cleanup() {
        System.out.println("Custom destroy method invoked.");
    }
}
```

---

### DisposableBean vs 다른 방식
| 방식                   | 특징                                                                                       | 사용 시기                   |
|------------------------|------------------------------------------------------------------------------------------|--------------------------|
| `DisposableBean`       | 인터페이스를 구현하므로 Spring 의존성이 강합니다.                                               | 간단한 리소스 정리 작업     |
| `@PreDestroy`          | JSR-250 표준에 따라 인터페이스에 의존하지 않고 사용할 수 있습니다.                              | 표준화된 방법 선호 시       |
| `@Bean(destroyMethod)` | 메서드 이름을 명시적으로 설정할 수 있어 특정 초기화/소멸 로직을 분리 가능.                           | 명시적으로 설정 필요할 때    |

---

### Spring Boot 3.3.5와 관련된 변화
Spring Boot 3.x (Spring Framework 6.x 기반)에서 `DisposableBean` 자체는 그대로 사용할 수 있지만, 다음 사항에 주의해야 합니다:
1. **Jakarta 패키지로의 전환:** `@PreDestroy`와 같은 어노테이션은 `javax.annotation` 대신 `jakarta.annotation`으로 이동했습니다.
2. **기능적 대체:** 인터페이스를 구현하지 않고도 간단히 리소스를 정리할 수 있는 다른 방식들이 더 권장됩니다.

💡 **결론:** `DisposableBean`은 여전히 사용 가능하지만, 현대적인 Spring 애플리케이션에서는 `@PreDestroy` 또는 `@Bean(destroyMethod)` 방식을 사용하는 것이 더 일반적이고 권장됩니다.
