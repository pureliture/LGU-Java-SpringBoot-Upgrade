이 코드는 **템플릿 엔진**을 사용해 이메일을 생성하는 설정을 정의한 스프링 구성 메서드입니다. 아래는 코드의 주요 동작 원리, 클래스별 역할, 그리고 이 설정이 전체적으로 어떤 일을 하는지 상세히 분석한 내용입니다.

---

### 1. **코드 설명**
```java
@Bean
public MailBuilder mailBuilder(ApplicationContext applicationContext) {
    // 템플릿 엔진 생성
    TemplateEngine templateEngine = new TemplateEngine();

    // HTML 템플릿용 리졸버 생성
    SpringResourceTemplateResolver htmlTemplateResolver = new SpringResourceTemplateResolver();
    htmlTemplateResolver.setPrefix(mailSetting.getTemplatePrefix());
    htmlTemplateResolver.setSuffix(mailSetting.getTemplateSuffix());
    htmlTemplateResolver.setCheckExistence(true);
    htmlTemplateResolver.setTemplateMode(TemplateMode.HTML);
    htmlTemplateResolver.setCacheable(true);
    htmlTemplateResolver.setCharacterEncoding(StandardCharsets.UTF_8.name());
    htmlTemplateResolver.setApplicationContext(applicationContext);
    htmlTemplateResolver.setOrder(1);

    // HTML 리졸버를 템플릿 엔진에 추가
    templateEngine.addTemplateResolver(htmlTemplateResolver);

    // 문자열 템플릿용 리졸버 생성
    StringTemplateResolver templateResolver = new StringTemplateResolver();
    templateResolver.setTemplateMode(TemplateMode.HTML);
    templateResolver.setOrder(2);

    // 문자열 리졸버를 템플릿 엔진에 추가
    templateEngine.addTemplateResolver(templateResolver);

    // MailBuilder를 생성하여 반환
    return new MailBuilder(templateEngine);
}
```

#### 동작 개요
1. **템플릿 엔진 생성**: 
   - 이메일 내용 생성을 위해 `TemplateEngine`을 사용합니다.
   - 다양한 템플릿 리졸버를 통해 템플릿을 처리할 수 있습니다.

2. **HTML 템플릿 리졸버 설정 (`SpringResourceTemplateResolver`)**:
   - 이메일 HTML 콘텐츠를 처리하기 위해 템플릿 리졸버를 설정합니다.
   - 리소스 경로(prefix)와 파일 확장자(suffix)를 설정하며, HTML 형식으로 처리되도록 구성합니다.
   - 이 리졸버는 스프링 애플리케이션 컨텍스트에서 템플릿 파일을 찾습니다.

3. **문자열 템플릿 리졸버 설정 (`StringTemplateResolver`)**:
   - 간단한 문자열 템플릿(파일이 아닌 문자열로 제공된 템플릿)을 처리합니다.
   - HTML 형식으로 처리되며, 우선순위(order)가 HTML 리졸버보다 낮습니다.

4. **`MailBuilder` 생성**:
   - 설정된 `TemplateEngine`을 사용하여 이메일 템플릿을 생성 및 처리하는 `MailBuilder` 객체를 반환합니다.

---

### 2. **클래스별 역할 및 상세 설명**

#### **1) MailBuilder**
- **역할**:
  - 이메일 콘텐츠를 생성하는 데 사용되는 유틸리티 클래스입니다.
  - 주로 템플릿 엔진을 사용하여 이메일의 본문을 렌더링하는 기능을 제공합니다.

- **주요 구성**:
  - `MailBuilder`는 생성자에서 `TemplateEngine`을 주입받아 사용합니다.
  - 템플릿 파일이나 문자열 템플릿을 통해 이메일 콘텐츠를 생성할 수 있습니다.

---

#### **2) TemplateEngine**
- **출처**: Thymeleaf 라이브러리의 핵심 클래스.
- **역할**:
  - 템플릿 처리의 핵심 역할을 수행하며, 템플릿 파일에서 동적으로 콘텐츠를 생성합니다.
  - 여러 개의 템플릿 리졸버(Resolver)를 조합하여 템플릿 처리 우선순위를 설정할 수 있습니다.

- **주요 메서드**:
  - `addTemplateResolver(ITemplateResolver resolver)`: 템플릿 리졸버를 추가하여 템플릿을 처리할 수 있도록 설정.
  - 내부적으로 템플릿 파일이나 문자열에 데이터를 바인딩하고, 최종 HTML을 생성합니다.

---

#### **3) SpringResourceTemplateResolver**
- **출처**: Thymeleaf에서 제공하며, 스프링 리소스 경로를 지원하는 템플릿 리졸버.
- **역할**:
  - `application.yml`이나 설정 클래스에서 지정한 템플릿 파일 경로(`prefix`, `suffix`)를 기반으로, 템플릿을 로드합니다.
  - 스프링의 `ResourceLoader`를 활용하여 클래스패스 또는 파일 시스템 경로에서 템플릿 파일을 검색합니다.

- **주요 설정**:
  - `setPrefix(String prefix)`: 템플릿 파일 경로의 기본값(예: `classpath:/templates/`).
  - `setSuffix(String suffix)`: 템플릿 파일 확장자(예: `.html`).
  - `setTemplateMode(TemplateMode mode)`: 템플릿 처리 모드(HTML, XML 등).
  - `setCacheable(boolean cacheable)`: 템플릿 파일 캐싱 여부.
  - `setCharacterEncoding(String encoding)`: 템플릿의 문자 인코딩 설정.

---

#### **4) StringTemplateResolver**
- **출처**: Thymeleaf에서 제공하는 문자열 기반 템플릿 리졸버.
- **역할**:
  - 파일 경로가 아닌, 문자열로 제공된 템플릿을 처리합니다.
  - HTML, XML, 텍스트 등 다양한 템플릿 모드를 지원합니다.
  - 주로 테스트나 동적으로 생성된 템플릿 처리에 사용됩니다.

- **주요 설정**:
  - `setTemplateMode(TemplateMode mode)`: 템플릿 처리 모드 설정.
  - `setOrder(int order)`: 템플릿 리졸버의 우선순위 설정.

---

### 3. **전체적인 동작 과정**
1. **템플릿 리졸버 설정**:
   - `SpringResourceTemplateResolver`는 이메일 HTML 템플릿 파일을 처리하며, 파일 경로와 확장자를 통해 템플릿을 로드합니다.
   - `StringTemplateResolver`는 문자열로 제공된 템플릿을 처리하며, 동적으로 생성된 템플릿을 처리할 때 유용합니다.

2. **템플릿 엔진 초기화**:
   - `TemplateEngine`은 등록된 리졸버들을 순서대로 호출하여 적합한 리졸버를 찾고, 템플릿을 처리합니다.

3. **메일 콘텐츠 생성**:
   - `MailBuilder`는 주입받은 `TemplateEngine`을 사용하여 HTML 기반 이메일 콘텐츠를 생성합니다.

4. **이메일 발송**:
   - 생성된 HTML 콘텐츠는 이메일 전송 서비스(`JavaMailSender` 등)로 전달되어 최종적으로 사용자가 수신하게 됩니다.

---

### 4. **실제 활용 사례**
- **HTML 이메일 발송**:
  - HTML 파일 템플릿(`classpath:/templates/email-template.html`)을 사용하여 동적으로 데이터를 렌더링한 뒤, 이메일 본문으로 사용.

- **템플릿 캐싱**:
  - `htmlTemplateResolver.setCacheable(true)` 설정으로 템플릿 파일을 캐싱하여 성능을 향상.

- **동적 템플릿 처리**:
  - `StringTemplateResolver`를 사용해, 특정 조건에 따라 동적으로 생성된 템플릿 문자열을 처리.

---

### **결론**
이 코드는 **Thymeleaf** 템플릿 엔진과 리졸버를 활용하여 이메일 콘텐츠를 동적으로 생성하기 위한 설정입니다. `SpringResourceTemplateResolver`는 HTML 파일 템플릿을 처리하며, `StringTemplateResolver`는 문자열로 제공된 템플릿을 처리합니다. 이러한 설정을 통해 정적 또는 동적 이메일 콘텐츠를 유연하게 생성하고 발송할 수 있습니다. 😄
