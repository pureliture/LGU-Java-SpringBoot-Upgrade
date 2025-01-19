
### **Spring Boot 애플리케이션 실행 및 Application Context 구성 과정**

---

#### **1. 애플리케이션 시작 (`main` 메서드 호출)**

Spring Boot 애플리케이션의 실행은 `SpringApplication.run()` 메서드로 시작됩니다:
```java
@SpringBootApplication
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

- `SpringApplication.run()`은 애플리케이션의 초기화를 시작하고 Spring Boot의 모든 부트스트랩 과정을 관리합니다.

---

#### **2. SpringApplication 초기화**

Spring Boot의 `SpringApplication` 클래스는 애플리케이션의 초기화를 관리합니다:
1. **Environment 생성**:
   - 애플리케이션 환경(`Environment`) 객체를 생성하고, 시스템 속성, 환경 변수, 애플리케이션 프로퍼티 등을 로드합니다.
2. **Application Context 타입 결정**:
   - WebApplication인지 일반 Application인지 컨텍스트 타입을 결정합니다.
   - Web 애플리케이션의 경우 `AnnotationConfigServletWebServerApplicationContext`가 사용됩니다.
3. **SpringFactories 로딩**:
   - `spring.factories` 파일을 기반으로 **ApplicationListeners**, **ApplicationContextInitializers** 등을 로드합니다.

---

#### **3. Environment 준비**

Spring Boot는 실행 환경을 구성하기 위해 다음을 수행합니다:
1. **Properties 파일 로드**:
   - `application.properties` 또는 `application.yml` 파일을 로드하여 `Environment`에 추가.
   - 외부 설정(명령줄 매개변수, 환경 변수 등)도 포함됩니다.
2. **ConfigurationProperties 바인딩**:
   - `@ConfigurationProperties`로 정의된 클래스에 프로퍼티를 바인딩하여 POJO 객체를 생성합니다.
   - 이 과정에서 `Binder`가 사용됩니다.

---

#### **4. Application Context 생성 및 준비**

1. **Application Context 생성**:
   - `SpringApplication`은 지정된 컨텍스트 유형(Web, Non-Web)에 따라 Application Context를 생성합니다.
   - `GenericApplicationContext` 또는 `WebApplicationContext`가 생성됩니다.

2. **Bean Factory 생성**:
   - `DefaultListableBeanFactory`가 생성되며, 모든 Bean의 생성과 관리를 담당합니다.

---

#### **5. Spring Boot Configuration Class 스캔 및 등록**

1. **`@ComponentScan` 실행**:
   - `@SpringBootApplication` 또는 `@ComponentScan`에 지정된 패키지를 스캔하여 다음을 발견합니다:
     - `@Component`, `@Service`, `@Repository`, `@Controller`
     - `@Configuration` 클래스
   - 발견된 클래스는 Bean으로 등록됩니다.

2. **Bean Definition 생성**:
   - 각 클래스의 정보를 기반으로 **Bean Definition**이 생성됩니다.
   - 이 과정에서 Bean의 의존성 주입 관계가 파악됩니다.

---

#### **6. Bean 등록 및 초기화**

1. **Bean 등록**:
   - `@Configuration` 클래스에서 선언된 `@Bean` 메서드를 실행하여 Bean을 생성하고 등록합니다.
   - `@Component`로 스캔된 클래스도 Bean으로 등록됩니다.

2. **Bean 초기화**:
   - Bean의 초기화 단계에서 다음 작업이 수행됩니다:
     - `@PostConstruct` 메서드 실행.
     - `InitializingBean` 인터페이스 구현체의 `afterPropertiesSet()` 호출.
     - 커스텀 `BeanPostProcessor` 실행.

---

#### **7. AOP 및 Proxy Bean 생성**

Spring AOP는 다음 단계에서 적용됩니다:
1. **Aspect 정의 발견**:
   - `@EnableAspectJAutoProxy`로 활성화된 경우, `@Aspect`로 선언된 클래스를 스캔합니다.
2. **Proxy 생성**:
   - AOP 대상 Bean에 대해 프록시 객체가 생성됩니다.
   - 프록시는 실제 Bean 호출 전에 AOP 어드바이스를 실행합니다.

---

#### **8. Web Application Context 구성 (Web 환경인 경우)**

1. **내장 서블릿 컨테이너 초기화**:
   - Tomcat, Jetty, Undertow와 같은 내장 서버가 초기화됩니다.
2. **DispatcherServlet 등록**:
   - Spring MVC 요청을 처리하는 `DispatcherServlet`이 서블릿 컨텍스트에 등록됩니다.
3. **핸들러 매핑 및 어댑터 등록**:
   - `RequestMappingHandlerMapping`과 `RequestMappingHandlerAdapter`가 등록되어 컨트롤러 매핑을 처리합니다.

---

#### **9. Event 발생**

Spring은 특정 시점마다 Application Event를 발생시킵니다:
- `ApplicationStartingEvent`: 애플리케이션이 시작되었을 때.
- `ApplicationReadyEvent`: 애플리케이션이 완전히 초기화되었을 때.

---

#### **10. ApplicationRunner 및 CommandLineRunner 실행**

Spring Boot는 `ApplicationRunner`와 `CommandLineRunner` 인터페이스를 구현한 Bean을 찾아 실행합니다:
```java
@Component
public class MyRunner implements ApplicationRunner {
    @Override
    public void run(ApplicationArguments args) {
        System.out.println("Application is ready!");
    }
}
```

---

### **Application Context 구성 전체 요약**

1. **SpringApplication.run()** 호출.
2. **Environment 생성**: 프로퍼티 로드 및 바인딩.
3. **Application Context 생성**:
   - BeanFactory 초기화.
4. **Component 스캔 및 Bean 등록**:
   - `@Component`, `@Configuration` 클래스 스캔 및 등록.
5. **Bean 초기화**:
   - `@PostConstruct`, `InitializingBean`, BeanPostProcessor 호출.
6. **AOP 프록시 생성**:
   - `@Aspect`가 적용된 Bean에 대해 프록시 생성.
7. **Web 환경 구성** (WebApplicationContext):
   - DispatcherServlet, 핸들러 매핑 등 등록.
8. **애플리케이션 준비 완료 이벤트 발생**.
9. **ApplicationRunner 및 CommandLineRunner 실행**.
