## Spring Boot 애플리케이션 실행 및 Application Context 구성 과정

### 1. 애플리케이션 시작(`main` 메소드 || @SpringBootTest 실행)

### 2. SpringApplication 초기화
1. Environment 생성
2. Application Context 타입 결정
- 일반 Application
3. SpringFactories 로딩:
- `org.springframework.boot.autoconfigure.AutoConfiguration.imports`에 정의된 클래스 로드
     - `EventuateInspectorConfig`
	 - `KafkaStreamInspectorConfig`
	 - `TopicInspectorConfig`

### 3. Environment 준비
1. Properties 파일 로드
- `application.yml` 파일을 로드하여 `Environment`에 추가.
2. ConfigurationProperties 바인딩
- `TopicInspectorProperties`
- `InspectorReaderProperties`(변수가 정의되어있음)
	- `(상속)SubscribeInspectorProperties`
	- `(상속)PublishInspectorProperties`

### 4. Application Context 생성 및 준비
- `GenericApplicationContext` 생성
- `DefaultListableBeanFactory` 생성

### 5. Spring Boot Configuration Class 스캔 및 등록
1) `TopicInspectorConfig` 스캔
- `spring.profiles.active`에 따라 활성화 여부가 달라짐

2) `KafkaStreamInspectorConfig`, `EventuateInspectorConfig` 스캔
> `@AutoConfigureAfter({TopicInspectorConfig.class})`
- `KafkaStreamInspectorConfig`는 `org.springframework.cloud.stream.binding` 관련 Bean 과 클래스가 정상적으로 존재해야만 활성화
	- `spring-cloud-stream, spring-cloud-stream-binder-kafka` 의존성이 포함되어 있지 않은 경우
	- `application.yml`에 Spring Cloud Stream 바인딩 설정이 명시되지 않은 경우
		- 응용 레포지토리(Ex. nuxx-svc-config)의 `application.yml` 파일에 명시되어 있음
- `EventuateInspectorConfig`는 Eventuate 관련 클래스가 클래스패스에 존재해야 활성화
	- `wafful-eventuate` 의존성이 포함되어 있지 않은 경우
	
### 6. Bean 등록 및 초기화
- `TopicInspectorConfig`
	- `SubscribeInspector` Bean 등록 및 초기화
	- `PublishInspector` Bean 등록 및 초기화
	- `AllowedTopicsReader` Bean 등록 및 초기화
- `KafkaStreamInspectorConfig`
	- (`AllowedTopicsReader` Bean 등록 성공 시)`KafkaStreamInspectorAspector` Bean 등록 및 초기화
- `EventuateInspectorConfig`
	- (`AllowedTopicsReader` Bean 등록 성공 시)`SubscribeInspectorAspect` Bean 등록 및 초기화
	- (`AllowedTopicsReader` Bean 등록 성공 시)`PublishInspectorAspect` Bean 등록 및 초기화

### 7. AOP(`@Aspect`) 및 Proxy Bean 생성
- `KafkaStreamInspectorAspector` 프록시 객체 생성
- `PublishInspectorAspect` 프록시 객체 생성
- `SubscribeInspectorAspect` 프록시 객체 생성

### 8. 애플리케이션 준비 완료 이벤트 발생
### 9. ApplicationRunner 및 CommandLineRunner 실행
