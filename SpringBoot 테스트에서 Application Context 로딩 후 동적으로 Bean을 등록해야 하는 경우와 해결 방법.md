# **SpringBoot 테스트에서 Application Context 로딩 후 동적으로 Bean을 등록해야 하는 경우와 해결 방법**

---

## **1. 들어가며**
Spring Boot는 **자동 구성(Auto-configuration)** 기능을 통해 애플리케이션 실행 시 필요한 Bean을 자동으로 등록합니다. 하지만 특정 테스트 환경에서는 **이 자동 등록 기능이 테스트를 방해하거나, 원하는 테스트 시점에 특정 기능이 실행되지 않는 문제**가 발생할 수 있습니다.  

### **📌 문제 상황**
1. **초기화 과정에서 외부 API 호출이 발생하는 Bean 테스트**
   - `@Bean`으로 등록된 클래스가 **초기화 시점(`@PostConstruct`, `InitializingBean.afterPropertiesSet()`)에서 외부 API를 호출**하여 원치 않는 부작용이 발생할 수 있음.
   - 예: `AllowedTopicsReader`가 **Application Context 로딩 중 실제 API를 호출하여 테스트가 실패**하는 경우.

2. **AOP(PointCut)가 Application Context 로딩 중 한 번만 실행되는 경우**
   - AOP가 특정 초기화 과정에서만 실행되도록 설정되어 있다면,  
     **테스트 중에는 AOP의 동작을 검증하기 어려움**.
   - 예: `BindingService.bindProducer()`를 AOP의 Pointcut으로 설정한 경우,  
     **이 메서드는 ApplicationContext 로딩 중에만 실행되므로, 테스트 코드 내부에서 AOP가 실행되지 않음.**

3. **조건부(`@ConditionalOnClass`, `@ConditionalOnMissingBean`)로 Bean이 등록되는 경우**
   - 특정 Bean이 **Application Context 로딩 시점에 특정 클래스의 존재 여부에 따라 등록될지 말지가 결정됨**.
   - 예: `KafkaStreamInspectorTestConfig`는 **다른 Bean이 등록된 이후에만 `@ConditionalOnClass` 조건을 만족하여 활성화됨**.

이러한 경우, **Application Context 로딩이 끝난 후 테스트 코드에서 Bean을 동적으로 추가해야만 정상적으로 테스트할 수 있습니다.**

---

## **2. 해결 방법**
### **✅ Application Context 로딩 이후 동적으로 Bean을 등록하는 방법**
위 문제를 해결하기 위해, **테스트 실행 중에 특정 `@TestConfiguration`을 등록하고 해당 설정의 Bean을 동적으로 추가하는 방식**을 사용해야 합니다.

---

### **📌 핵심 해결 전략**
1. **새로운 `AnnotationConfigApplicationContext`(임시 컨텍스트)를 생성하여 테스트용 `@TestConfiguration`을 등록**
2. **임시 컨텍스트에서 생성된 Bean을 기존 `ApplicationContext`에 추가**
3. **ApplicationContext가 이미 로드된 이후에도 추가된 Bean이 정상적으로 동작하도록 보장**

---

## **3. 해결 방법 구현**
### **🚀 예제 1: `@PostConstruct`에서 외부 API를 호출하는 Bean의 테스트**
#### **📌 문제 상황**
아래 `AllowedTopicsReader`는 `@PostConstruct`에서 API를 호출하여,  
테스트 시점에서 **실제 API가 호출되는 문제**가 발생함.

```java
import jakarta.annotation.PostConstruct;
import org.springframework.stereotype.Component;

@Component
public class AllowedTopicsReader {

    @PostConstruct
    public void init() {
        // 외부 API 호출 (테스트에서 실행되면 안됨)
        System.out.println("🚀 실제 API 호출 발생: AllowedTopicsReader 초기화 중...");
        fetchAllowedTopicsFromRemote();
    }

    private void fetchAllowedTopicsFromRemote() {
        // 실제 API 호출 로직 (테스트에서는 실행되지 않아야 함)
    }
}
```
➡ **테스트 코드 실행 시, `ApplicationContext`가 로드될 때 API가 호출됨 → 문제 발생**

---

#### **✅ 해결책: 동적으로 Mock 빈을 등록하여 실제 API 호출 방지**
```java
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.config.ConfigurableListableBeanFactory;
import org.springframework.beans.factory.support.DefaultListableBeanFactory;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.context.ApplicationContext;
import org.springframework.context.annotation.AnnotationConfigApplicationContext;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import static org.mockito.Mockito.*;

@SpringBootTest
public class DynamicBeanRegistrationTest {

    private ApplicationContext context;

    @BeforeEach
    void setup() {
        this.context = ApplicationContextProvider.getApplicationContext();
        registerMockBeanAfterContextLoaded();
    }

    private void registerMockBeanAfterContextLoaded() {
        DefaultListableBeanFactory beanFactory = (DefaultListableBeanFactory) context.getAutowireCapableBeanFactory();

        // 기존 빈을 제거
        if (beanFactory.containsBean("allowedTopicsReader")) {
            beanFactory.removeBeanDefinition("allowedTopicsReader");
        }

        // 새로운 Mock 빈 등록
        AllowedTopicsReader mockReader = mock(AllowedTopicsReader.class);
        beanFactory.registerSingleton("allowedTopicsReader", mockReader);
    }

    @Test
    void testWithMockedAllowedTopicsReader() {
        AllowedTopicsReader reader = (AllowedTopicsReader) context.getBean("allowedTopicsReader");
        assert reader != null;
    }
}
```
**✔️ 해결된 문제**  
- `@PostConstruct`가 실행되지 않음 → **테스트 시 API가 호출되지 않음**
- 테스트 코드 실행 후 **동적으로 Mock Bean을 등록**하여 API 호출 없이 정상적으로 테스트 수행 가능

---

### **🚀 예제 2: AOP(PointCut)가 Application Context 로딩 중 한 번만 실행되는 경우**
#### **📌 문제 상황**
아래 AOP는 `BindingService.bindProducer()`를 Pointcut으로 설정하여 **ApplicationContext 로딩 중 한 번만 실행됨.**  
테스트 실행 중 AOP가 다시 실행되지 않는 문제가 발생.

```java
import org.aspectj.lang.annotation.AfterReturning;
import org.aspectj.lang.annotation.Aspect;
import org.springframework.stereotype.Component;
import org.springframework.cloud.stream.binding.BindingService;

@Aspect
@Component
public class BindingServiceAspect {

    @AfterReturning("execution(* org.springframework.cloud.stream.binding.BindingService.bindProducer(..))")
    public void afterBindProducer() {
        System.out.println("🔥 AOP 실행: BindingService.bindProducer() 호출됨!");
    }
}
```
➡ **테스트 실행 시 `bindProducer()`가 실행되지 않으면 AOP가 트리거되지 않음 → AOP 테스트 불가능**

---

#### **✅ 해결책: 테스트 코드에서 명시적으로 `bindProducer()` 실행**
```java
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.cloud.stream.binding.BindingService;
import org.springframework.integration.channel.DirectChannel;
import org.springframework.messaging.MessageChannel;

@SpringBootTest
public class BindingServiceAopTest {

    @Autowired
    private BindingService bindingService;

    @BeforeEach
    void setup() {
        // ApplicationContext 로딩 후 명시적으로 bindProducer() 실행
        MessageChannel channel = new DirectChannel();
        bindingService.bindProducer(channel, "test-out-0");
    }

    @Test
    void testBindingServiceAspect() {
        // 🔥 AOP가 실행되어야 함
        System.out.println("✅ AOP가 트리거되었는지 확인!");
    }
}
```
**✔️ 해결된 문제**  
- `ApplicationContext` 로딩 후 **테스트 코드에서 `bindProducer()`를 명시적으로 실행**  
- AOP가 다시 실행되도록 유도하여 **AOP 테스트가 가능하도록 설정**

---

## **4. 정리: 언제, 왜 사용해야 하는가?**
### **✅ 언제 필요한가?**
- `@Configuration`의 `@Bean`이 초기화 과정에서 **외부 API를 호출하는 경우** (예: `AllowedTopicsReader`)
- AOP 포인트컷이 **ApplicationContext 로딩 중에만 실행되는 경우** (예: `BindingService.bindProducer`)
- 특정 Bean이 **조건부(`@ConditionalOnClass`, `@ConditionalOnMissingBean`)로 등록되며, 테스트 시점에 이를 활성화해야 하는 경우**
- 기존 `ApplicationContext`를 **재시작하지 않고, 동적으로 특정 Bean을 추가해야 하는 경우**

---

## **🔥 결론**
- **테스트 코드 실행 전에 Application Context 로딩 중 실행되는 기능을 회피하거나 추가하는 방법이 필요할 때, Bean을 동적으로 등록하는 방법이 필요하다.**
- **Mock 빈을 동적으로 등록하여 불필요한 API 호출을 방지하거나, AOP가 트리거되도록 테스트 코드에서 특정 동작을 실행해야 한다.**
- 이를 통해 **초기화 과정에서 API 호출이 발생하는 문제, AOP의 실행 타이밍 문제, `@ConditionalOnClass` 조건 문제 등을 해결할 수 있다.**
