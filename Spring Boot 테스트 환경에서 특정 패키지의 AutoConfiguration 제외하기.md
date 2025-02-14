# **Spring Boot 테스트 환경에서 특정 패키지의 AutoConfiguration 제외하기**

## **1. 개요**
Spring Boot는 자동 구성을 통해 애플리케이션 실행 시 필요한 빈(Bean)을 자동으로 등록합니다. 그러나 테스트 환경에서는 특정 자동 구성(AutoConfiguration) 클래스가 불필요하거나, 오히려 테스트 실행에 방해가 될 수 있습니다.  
이를 해결하기 위해 **AutoConfigurationImportFilter** 인터페이스를 활용하여 특정 패키지의 자동 구성을 제외하는 방법을 소개합니다.

---

## **2. 목적**
Spring Boot 테스트 환경에서 **특정 패키지에 속한 자동 구성 클래스를 제외**하여 **테스트 컨텍스트의 불필요한 구성 등록을 방지**하고, **테스트 실행 속도를 향상**시키며, **불필요한 의존성으로 인한 충돌을 방지**하는 것이 목표입니다.

### **적용 대상**
- 테스트 실행 시 특정 자동 구성을 로드하지 않도록 제한하려는 경우
- 특정 라이브러리의 자동 구성이 테스트 환경에서 필요하지 않은 경우
- 특정 패키지의 자동 구성이 테스트 실행을 방해하는 경우

---

## **3. 구현 방법**
Spring Boot에서는 `AutoConfigurationImportFilter` 인터페이스를 구현하여 특정 패키지 경로의 자동 구성을 제외할 수 있습니다.

### **3.1. `AutoConfigurationImportFilter` 구현**
다음과 같이 특정 패키지에 속하는 자동 구성을 제외하는 `AutoConfigurationImportFilterForSpringBootTest` 클래스를 구현합니다.

```java
package com.example.config;

import org.springframework.boot.autoconfigure.AutoConfigurationImportFilter;
import org.springframework.boot.autoconfigure.AutoConfigurationMetadata;

import java.util.Arrays;
import java.util.List;

/**
 * 특정 패키지의 자동 구성(AutoConfiguration) 클래스를 테스트 실행 시 제외하는 필터 클래스입니다.
 */
public class AutoConfigurationImportFilterForSpringBootTest implements AutoConfigurationImportFilter {

    /**
     * 자동 구성에서 제외할 패키지 및 클래스 목록
     */
    private static final List<String> EXCLUDED_PACKAGE = Arrays.asList(
            "io.eventuate",  // 오픈소스 Eventuate 패키지
            "com.example.messaging.producer",
            "com.example.messaging.consumer",
            "com.example.config.web.EventuateConfig",
            "com.example.config.web.TestMessageConfig"
    );

    /**
     * 특정 자동 구성 클래스를 필터링하여 제외합니다.
     *
     * @param autoConfigurationClasses 자동 구성 후보 클래스의 전체 클래스명 배열
     * @param autoConfigurationMetadata 자동 구성 클래스와 관련된 메타데이터
     * @return {@code true}이면 해당 클래스가 포함되며, {@code false}이면 제외됩니다.
     */
    @Override
    public boolean[] match(String[] autoConfigurationClasses, AutoConfigurationMetadata autoConfigurationMetadata) {
        boolean[] matchResults = new boolean[autoConfigurationClasses.length];

        for (int i = 0; i < autoConfigurationClasses.length; i++) {
            String className = autoConfigurationClasses[i];
            if (className == null) {
                matchResults[i] = true; // 기본적으로 포함
            } else if (EXCLUDED_PACKAGE.stream().anyMatch(className::startsWith)) {
                matchResults[i] = false; // 제외 대상
            } else {
                matchResults[i] = true; // 포함 대상
            }
        }
        return matchResults;
    }
}
```

---

### **3.2. `spring.factories` 설정**
위에서 만든 `AutoConfigurationImportFilter`를 Spring Boot에서 자동으로 감지하고 적용하려면, `spring.factories` 파일을 사용해야 합니다.

`src/test/resources/META-INF/spring.factories` 파일을 생성하고 다음 내용을 추가합니다.

```
org.springframework.boot.autoconfigure.AutoConfigurationImportFilter=\
com.example.config.AutoConfigurationImportFilterForSpringBootTest
```

이 설정을 통해 Spring Boot는 테스트 실행 시 `AutoConfigurationImportFilterForSpringBootTest`를 적용하여 특정 패키지에 속한 자동 구성 클래스를 제외합니다.

---

## **4. 기대 효과**
위와 같은 설정을 적용함으로써 다음과 같은 효과를 기대할 수 있습니다.

### **4.1. 불필요한 자동 구성 제거**
- 테스트 실행 시 `@AutoConfiguration`이 적용된 특정 패키지의 구성이 로드되지 않음
- 테스트 컨텍스트가 가벼워져 실행 속도가 향상됨

### **4.2. 테스트 환경 충돌 방지**
- 특정 자동 구성 클래스가 포함될 경우, 기존 Bean과 충돌할 가능성이 있음
- 불필요한 자동 구성을 제외함으로써 이러한 충돌을 사전에 방지 가능

### **4.3. 테스트 성능 최적화**
- 자동 구성이 제외됨에 따라, 테스트 컨텍스트 로딩 시간이 단축됨
- 테스트 실행 시간이 단축되어 CI/CD 파이프라인에서도 유리함

---

## **5. 결론**
Spring Boot 테스트 환경에서는 불필요한 자동 구성을 제외하여 **테스트 성능을 최적화**하고, **불필요한 의존성 충돌을 방지**하는 것이 중요합니다.  
이를 위해 `AutoConfigurationImportFilter`를 활용하면, 특정 패키지의 자동 구성을 **테스트 컨텍스트에서 제외**할 수 있습니다.

### **요약**
- `AutoConfigurationImportFilter`를 구현하여 특정 패키지의 자동 구성을 제외
- `spring.factories`에 필터를 등록하여 Spring Boot 테스트 컨텍스트에 반영
- 불필요한 구성 제외로 인해 **테스트 속도 향상, 충돌 방지, 테스트 환경 최적화** 효과 기대

이 방법을 적용하면, **보다 효율적인 Spring Boot 테스트 환경을 구축**할 수 있습니다. 🚀
