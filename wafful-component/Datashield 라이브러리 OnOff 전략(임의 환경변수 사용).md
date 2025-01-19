1. 임의 환경변수와 ExecutionCondition를 활용한 Datashield 활성화/비활성화테스트 환경 파일:src/test/resources/datashield-activation-application.ymlsrc/test/resources/datashield-deactivation-application.ymlSpring Configuration 클래스:DatashieldActivationTestConfigDatashieldDeactivationTestConfig테스트 실행 조건:DatashieldActivateTestExecutionConditionDatashieldDeactivateTestExecutionCondition.yml 파일 분석datashield-activation-application.yml:wafful-datashield:
  enabled: true
  ds-config-home: classpath:/config/
  datatype-config: classpath:/config/datashield-activation-datatype.yml
  application:
    system-code: NUCdatashield-deactivation-application.yml:wafful-datashield:
  enabled: true
  datatype-config: classpath:/config/datashield-deactivate/datashield-datatype.yml
  application:
    system-code: NUC이 두 .yml 파일은 test.datashield.activate 값을 기반으로 Spring의 @ConditionalOnProperty 및 ExecutionCondition에 의해 활성화/비활성화 동작을 제어합니다.2. ExecutionCondition를 활용한 테스트 제어활성화된 조건 구현: DatashieldActivateTestExecutionConditionimport org.junit.jupiter.api.extension.ConditionEvaluationResult;
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
}비활성화된 조건 구현: DatashieldDeactivateTestExecutionConditionimport org.junit.jupiter.api.extension.ConditionEvaluationResult;
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
}3. Spring Configuration 클래스활성화된 상태에서의 설정: DatashieldActivationTestConfigimport org.springframework.boot.autoconfigure.condition.ConditionalOnProperty;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Conditional(DatashieldActivationTestConfig.OnProperty.class)
@ConditionalOnProperty(prefix = DatashieldProperties.PREFIX, name = "enabled", havingValue = "true")
@EnableConfigurationProperties(value = {DatashieldProperties.class})
@Configuration
public class DatashieldActivationTestConfig {

    @Bean
    public DatashieldClient datashieldClient() {
        return new DatashieldClientImpl(); // 실제 클라이언트 구현체
    }

public static class OnProperty implements Condition {

        @Override
        public boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata) {

            Environment environment = context.getEnvironment();

            try {
                String datashieldActivation = environment.getProperty("test.datashield.activate");

                return StringUtils.equals("true", datashieldActivation);

            } catch (Exception e) {
                log.warn("OnProperty error {}", e.getMessage());
                return false;
            }
        }
    }
}비활성화된 상태에서의 설정: DatashieldDeactivationTestConfigimport org.springframework.boot.autoconfigure.condition.ConditionalOnProperty;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Slf4j
@Configuration
@Conditional(DatashieldDeactivationTestConfig.OnProperty.class)
@ConditionalOnProperty(prefix = DatashieldProperties.PREFIX, name = "enabled", havingValue = "true")
@EnableConfigurationProperties(value = {DatashieldProperties.class})
public class DatashieldDeactivationTestConfig {

    @Bean
    public MockDatashieldClient mockDatashieldClient() {
        return new MockDatashieldClient(); // Mock 클라이언트 구현체
    }

    public static class OnProperty implements Condition {

        @Override
        public boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata) {

            Environment environment = context.getEnvironment();

            try {
                String datashieldActivation = environment.getProperty("test.datashield.activate");

                return Boolean.FALSE.equals(StringUtils.equals("true", datashieldActivation));

            } catch (Exception e) {
                log.warn("OnProperty error {}", e.getMessage());
                return false;
            }
        }

    }
}4. 테스트 코드활성화된 테스트: DatashieldInitializerSpringBootTestimport org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.springframework.boot.test.context.SpringBootTest;

@ExtendWith(DatashieldActivateTestExecutionCondition.class)
@ContextConfiguration(classes = DatashieldActivationTestConfig.class)
@SpringBootTest(properties = "spring.config.location=classpath:datashield-activation-application.yml")
public class DatashieldInitializerSpringBootTest {

    @Test
    void testDatashieldActivation() {
        System.out.println("Datashield is activated and tested.");
    }
}비활성화된 테스트: DatashieldDeactivationTestimport org.junit.jupiter.api.Test;
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
}5. 요약활성화/비활성화 전략:환경 파일 (.yml):test.datashield.activate 값을 통해 활성화(true)와 비활성화(false)를 구분.Spring의 @ConditionalOnProperty 및 테스트 조건에서 해당 값을 참조.ExecutionCondition:JUnit 5의 ExecutionCondition을 활용하여 조건에 따라 테스트를 활성화 또는 비활성화.활성화: DatashieldActivateTestExecutionCondition.비활성화: DatashieldDeactivateTestExecutionCondition.Spring Configuration:@ConditionalOnProperty를 통해 활성화(true)와 비활성화(false) 상태에서 다른 Bean을 주입.테스트 코드:활성화된 상태와 비활성화된 상태를 각각 검증하기 위해 분리된 테스트 클래스를 작성.6. 문제점Datashield 활성화 여부에 따른 테스트 Ignore 여부 지정 시 ExecutionCondition 의 대안으로 알려진 기능이 테스트 Ignore 처리를 정상적으로 수행해주지 않음@EnabledIF, @DisabledIf@IfProfileValue@EnabledIfSystemProperty, @DisabledIfSystemPropertyExecutionCondition 사용 시 Ignore 처리가 가능하나 불필요하게 클래스가 추가되며 코드가 복잡해짐활성화/비활성화 각각에 대한 application.yml 파일 사용이 어려움spring.active.profiles를 활용하여 분리한 경우 application-${profile}.yml 규칙에 맞춰 파일 추가 시 자동으로 분리되나 프로파일을 사용하지 않을 경우 별도 매핑 필요@PropertySource + EnvironmentPostProcessor + YamlPropertySourceFactory -> 구현했으나 바인딩 실패@TestPropertySource -> 단독 사용 시 바인딩 실패@SpringBootTest(properties = "spring.profiles.active=datashield-activate") -> 바인딩 성공
