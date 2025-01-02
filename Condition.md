## Spring의 Condition 인터페이스
Condition은 Spring Framework에서 제공하는 조건부 Bean 등록을 구현하기 위한 핵심 인터페이스입니다.
이를 활용하면 특정 조건이 만족될 때에만 Bean을 등록하거나 활성화할 수 있습니다.

### Condition 인터페이스의 역할
- 조건부 로직: 특정 조건에 따라 Bean을 등록하거나 제외하는 논리를 작성.
- 유연한 Bean 관리: 환경 변수, 프로파일, 클래스 로딩 여부 등 다양한 요소에 따라 Bean 등록 제어.
- 런타임 조건 검증: 런타임에서 조건을 동적으로 평가.

### Condition의 메서드
1. matches
- 정의:
```java
boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata);
```
- 역할: Bean 등록 여부를 결정하는 조건 로직을 구현.
- 매개변수:
  - ConditionContext context:
    - Bean 등록 환경에 대한 정보를 제공.
    - 관련 클래스:
      - ConditionContext: 애플리케이션 컨텍스트, BeanFactory, 환경 변수 접근.
  - AnnotatedTypeMetadata metadata:
    - Bean에 선언된 어노테이션 메타데이터를 제공.
    - 관련 클래스:
      - AnnotatedTypeMetadata: 어노테이션의 존재 여부와 속성 정보 제공.
- 반환값:
  - true: 조건이 만족되어 Bean 등록.
  - false: 조건이 불만족하여 Bean 미등록.

### Condition 인터페이스 사용 방법
1. 간단한 조건 구현
아래는 특정 환경 변수(feature.enabled)가 true일 때만 Bean을 등록하는 예제입니다.

```java
import org.springframework.context.annotation.Condition;
import org.springframework.context.annotation.ConditionContext;
import org.springframework.core.type.AnnotatedTypeMetadata;
public class FeatureEnabledCondition implements Condition {
    @Override
    public boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata) {
        // 환경 변수 값을 읽음
        String featureEnabled = context.getEnvironment().getProperty("feature.enabled");
        return "true".equalsIgnoreCase(featureEnabled);
    }
}
```

2. 조건을 Bean 등록에 적용
@Conditional을 사용하여 특정 Bean 등록 시 조건부 로직을 추가할 수 있습니다.

```java
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Conditional;
public class AppConfig {
    @Bean
    @Conditional(FeatureEnabledCondition.class)
    public MyService myService() {
        return new MyService();
    }
}
```

3. 어노테이션 기반 조건 처리
다음은 특정 어노테이션이 선언된 경우에만 Bean을 등록하는 예제입니다.

```java
public class AnnotationBasedCondition implements Condition {
    @Override
    public boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata) {
        // 특정 어노테이션이 있는지 확인
        return metadata.isAnnotated("org.springframework.stereotype.Controller");
    }
}
```

### ConditionContext와 AnnotatedTypeMetadata 역할 상세
#### ConditionContext
- ConditionContext는 조건부 로직 구현 시 필요한 환경 정보를 제공합니다.
- 주요 정보:
  - getEnvironment(): 환경 변수 및 프로파일.
  - getBeanFactory(): Bean 정의 및 상태.
  - getRegistry(): Bean 등록/제거.

#### AnnotatedTypeMetadata
- AnnotatedTypeMetadata는 어노테이션 정보를 제공하여 Bean 클래스나 메서드에 선언된 어노테이션을 기반으로 조건을 평가할 수 있습니다.
- 주요 메서드:
  - isAnnotated(String annotationName): 어노테이션 존재 여부 확인.
  - getAnnotationAttributes(String annotationName): 어노테이션 속성 값 조회.

### 조건부 로직 구현 예제
**예제 1: 특정 환경 변수 기반 Bean 등록**

```java
public class EnvCondition implements Condition {
    @Override
    public boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata) {
        String env = context.getEnvironment().getProperty("app.env");
        return "production".equalsIgnoreCase(env);
    }
}
```

**예제 2: 어노테이션 조건 기반 Bean 등록**

```java
public class AnnotationCondition implements Condition {
    @Override
    public boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata) {
        return metadata.isAnnotated("org.springframework.stereotype.Service");
    }
}
```

**예제 3: 특정 Bean 존재 여부 기반 조건**

```java
public class BeanPresenceCondition implements Condition {
    @Override
    public boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata) {
        return context.getBeanFactory().containsBeanDefinition("requiredBean");
    }
}
```

### Condition 인터페이스의 확장성과 유용성
- **확장성**: 환경 변수, 어노테이션, Bean 존재 여부, 파일 시스템 상태 등 다양한 조건을 평가 가능.
- **유용성**: 애플리케이션의 다양한 요구 사항을 충족하는 동적 Bean 구성 가능.

### 요약
- Condition 인터페이스는 조건부 Bean 등록을 위한 핵심 인터페이스.
- ConditionContext는 환경 정보와 Bean 상태를 제공.
- AnnotatedTypeMetadata는 어노테이션 정보를 기반으로 조건을 처리.
- matches 메서드를 통해 유연한 조건 로직 구현 가능.

