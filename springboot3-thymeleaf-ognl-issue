1. 문제 상황

Spring Boot 2.7.5에서 Spring Boot 3.3.5로 프레임워크를 업그레이드하는 과정에서 NoClassDefFoundError: ognl/PropertyAccessor 오류가 발생했습니다.
이 문제는 Thymeleaf 템플릿 엔진이 OGNL(Object-Graph Navigation Language)을 의존하지 않았던 Spring Boot 2.7.5와 달리, Spring Boot 3.3.5의 Thymeleaf(버전 3.1.x 이상)에서 OGNL 의존성이 선택적으로 추가되며 발생한 것으로 확인되었습니다.

2. 문제 원인



Spring Boot 2.7.5의 동작:

Thymeleaf는 Spring EL(Spring Expression Language)을 기본 표현식 언어로 사용하며, OGNL은 포함되지 않았습니다.
따라서 OGNL 의존성이 없더라도 정상적으로 동작했습니다.



Spring Boot 3.3.5로 업그레이드 후:

Thymeleaf 3.1.x 이상에서는 OGNL이 선택적으로 사용되며, OGNL 의존성이 없으면 관련 클래스 로딩 오류가 발생할 수 있습니다.
기존 프로젝트는 OGNL을 명시적으로 추가하지 않았기 때문에 Spring Boot 3.3.5 환경에서 오류가 발생했습니다.





3. 해결 방안

문제를 해결하기 위해 두 가지 방법을 선택할 수 있습니다:


방법 1: OGNL 제거 및 Spring EL 사용

Thymeleaf에서 OGNL을 완전히 제거하고 Spring EL을 사용하는 방법입니다.
Spring EL은 Spring Context와 통합이 자연스럽고 추가 의존성이 필요하지 않으므로 안정적인 선택입니다.
1.1. Spring EL 활성화: Spring EL을 Thymeleaf의 기본 표현식 언어로 활성화합니다.


application.yml:

spring: thymeleaf: enable-spring-el-compiler: true




Java Config:

SpringTemplateEngine templateEngine = new SpringTemplateEngine(); 
templateEngine.setEnableSpringELCompiler(true);




1.2. OGNL 의존성 제거: pom.xml에서 OGNL 의존성을 제거합니다.

<dependency>
    <groupId>ognl</groupId>
    <artifactId>ognl</artifactId>
</dependency>


1.3. TemplateEngine에서 SpringTemplateEngine으로 변경: 기존 프레임워크에서 TemplateEngine 클래스를 사용 중이라면, 이를 SpringTemplateEngine으로 변경해야 Spring EL 설정이 적용됩니다.


기존 코드 (TemplateEngine 사용):

TemplateEngine templateEngine = new TemplateEngine();
templateEngine.setTemplateResolver(templateResolver);




변경된 코드 (SpringTemplateEngine 사용):

SpringTemplateEngine templateEngine = new SpringTemplateEngine(); 
templateEngine.setTemplateResolver(templateResolver); templateEngine.setEnableSpringELCompiler(true);




장점:

추가적인 의존성을 관리할 필요가 없습니다.
Spring Boot와 Thymeleaf의 기본 설정을 활용하여 안정적인 동작을 보장합니다.



방법 2: OGNL 의존성 추가

Thymeleaf에서 OGNL을 계속 사용하려면 프로젝트에 OGNL 의존성을 명시적으로 추가합니다.
2.1. OGNL 의존성 추가: pom.xml에 OGNL 의존성을 추가합니다.

<dependency>
    <groupId>ognl</groupId>
    <artifactId>ognl</artifactId>
    <version>3.2.11</version>
</dependency>


2.2. 의존성 충돌 점검: Maven 의존성 트리(mvn dependency:tree)를 사용하여 OGNL이 다른 라이브러리와 충돌하지 않는지 확인합니다.
장점:

기존 OGNL 기반 템플릿 코드를 수정할 필요가 없습니다.
OGNL의 고급 기능을 계속 사용할 수 있습니다.



4. 비교 및 권장 사항




방법
장점
단점




Spring EL 사용
- 추가 의존성 불필요- Spring Boot의 기본 설정 활용- 안정적인 동작 보장
- OGNL 기반의 기능 사용 불가


OGNL 추가
- 기존 OGNL 기반 코드 수정 불필요- OGNL 기능 활용 가능
- 추가 의존성 필요- 의존성 관리 및 충돌 가능성 존재



권장 방안:


Spring EL 사용이 기본적으로 권장됩니다. Spring Boot와 Thymeleaf의 기본 동작에 따라 안정적으로 설정할 수 있으며, 추가 의존성 관리가 필요 없습니다.
OGNL이 반드시 필요한 경우에만 명시적으로 추가합니다.



5. 최종 결론

Spring Boot 3.3.5 업그레이드 과정에서 발생한 OGNL 관련 문제는 다음 두 가지 방법 중 하나로 해결할 수 있습니다:

OGNL을 제거하고 Spring EL을 활성화합니다.

기존 TemplateEngine을 SpringTemplateEngine으로 변경하고, Spring EL 컴파일러를 활성화해야 합니다.


OGNL 의존성을 추가하여 오류를 방지합니다.
