### **SpringBootTest 시 ApplicationContext 초기화 이전에 MockedStatic을 등록할 경우 발생하는 문제와 해결 방안**

---

### **1. 요구사항**

SpringBoot 환경에서 `@SpringBootTest`를 사용하여 테스트를 실행할 때, **ApplicationContext 초기화 중 호출되는 static 메서드를 Mock 처리**해야 하는 요구사항이 있었습니다. 이 요구사항의 배경과 이유는 다음과 같습니다:

1. **ApplicationContext 초기화 중 외부 리소스 호출 차단**:
   - 특정 static 메서드(`PropertyUtil.getResource()`)가 ApplicationContext 초기화 과정에서 호출되며, 외부 API 호출이나 파일 시스템 접근과 같은 리소스 의존성을 포함하고 있었습니다.
   - 테스트 환경에서는 외부 호출을 차단하고, 반환값을 Mock 처리하여 제어된 테스트 환경을 구축하고자 했습니다.

2. **테스트의 독립성과 안정성 보장**:
   - 테스트 환경에서 외부 리소스 호출로 인해 발생할 수 있는 예외를 방지하고, 특정 입력값에 따라 반환값이 결정되는 Mock 상태를 유지해야 했습니다.

3. **테스트와 ApplicationContext 초기화의 통합**:
   - ApplicationContext 초기화가 완료된 후에 Mock 처리된 메서드가 호출되는 것이 아니라, **ApplicationContext 초기화 이전에 Mock 상태가 적용되어야 하는 요구사항**이 있었습니다.

---

### **2. 문제 발견**

MockedStatic을 활용하여 static 메서드를 Mock 처리하는 과정에서 다음과 같은 문제가 발견되었습니다:

1. **ApplicationContext 초기화 중 MockedStatic 상태 불일치**:
   - MockedStatic은 Thread-local로 관리되며, 특정 스레드에서 생성된 Mock 상태는 해당 스레드에서만 유효합니다.
   - ApplicationContext 초기화는 테스트 실행과 동일한 스레드에서 수행되지만, Thread-local 상태와 코드에서 관리하는 MockedStatic 객체(`mockedStatic` 변수)가 동기화되지 않아, **`mockedStatic`이 `null` 상태로 남게 되는 문제**가 발생했습니다.

2. **MockedStatic 중복 생성 오류**:
   - 테스트 실행 도중, 기존에 존재하는 MockedStatic 객체가 초기화되지 않은 상태에서 새로운 MockedStatic 객체를 생성하려 할 때, **`To create a new mock, the existing static mock registration must be deregistered`** 오류가 발생했습니다.
   - 이는 Thread-local로 관리되는 MockedStatic 상태를 명확히 초기화하지 않고 새로운 Mock을 생성하려 했기 때문입니다.

3. **MockedStatic close 호출 누락**:
   - MockedStatic이 실제로 Thread-local에 존재하더라도, `mockedStatic` 변수가 `null` 상태로 남아 `close()` 호출이 누락되는 경우가 발생했습니다.
   - 이는 Thread-local 상태와 `mockedStatic` 변수 간의 상태 불일치로 인한 문제였습니다.

---

### **3. 원인 분석**

#### **MockedStatic이 close되지 않는 이유**

1. **Thread-local에서의 Mock 상태 관리**:
   - MockedStatic은 Thread-local로 관리되므로, Mock 객체가 생성된 스레드에서만 상태가 유지됩니다.
   - 테스트 실행 및 ApplicationContext 초기화가 동일한 스레드에서 진행되지만, Thread-local에 남아 있는 기존 Mock 상태가 초기화되지 않으면 새로운 Mock을 생성하려 할 때 충돌이 발생합니다.

2. **Mock 상태 초기화 실패**:
   - 기존 Mock 상태를 명확히 초기화하지 않으면, Mock 상태가 남아 있어 중복 Mock 생성 문제를 일으킵니다.

3. **MockedStatic 변수와 Thread-local 상태 간 불일치**:
   - MockedStatic 변수(`mockedStatic`)는 Thread-local 상태와 동기화되지 않아, 실제로 Thread-local에 존재하는 Mock 상태와 관리되는 Mock 상태가 일치하지 않는 문제가 발생했습니다.

---

### **4. 해결 방안**

#### **1. Thread-local 상태 강제 초기화**
- 테스트 실행 및 종료 시 **`Mockito.framework().clearInlineMocks()`를 호출**하여 모든 Thread-local 상태를 초기화합니다.
- 이를 통해 Mock 중복 생성 문제를 방지할 수 있습니다.

#### **2. MockedStatic 상태 명확히 관리**
- MockedStatic 객체를 생성하기 전에 반드시 기존 상태를 초기화합니다.
- 테스트 종료 후 Mock 상태를 정리하며, Thread-local에 남아 있는 상태를 명확히 초기화합니다.

---

### **5. 최종 코드**

#### **MockStaticTestExecutionListener**

```java
package com.lguplus.ncube.config.web;

import com.lguplus.wafful.framework.util.PropertyUtil;
import org.mockito.MockedStatic;
import org.mockito.Mockito;
import org.springframework.test.context.TestContext;
import org.springframework.test.context.support.AbstractTestExecutionListener;

public class MockStaticTestExecutionListener extends AbstractTestExecutionListener {

    private static MockedStatic<PropertyUtil> mockedStatic;

    @Override
    public void beforeTestClass(TestContext testContext) {
        // 현재 Thread-local의 모든 Mock 상태 초기화
        Mockito.framework().clearInlineMocks();

        // MockedStatic 생성
        mockedStatic = Mockito.mockStatic(PropertyUtil.class);
        System.out.println("Created MockedStatic for PropertyUtil");

        // Mock 동작 설정
        mockedStatic.when(() -> PropertyUtil.getResource("test-url"))
                    .thenReturn(null);
    }

    @Override
    public void afterTestClass(TestContext testContext) {
        // MockedStatic 해제
        if (mockedStatic != null) {
            mockedStatic.close();
            mockedStatic = null;
            System.out.println("Closed MockedStatic for PropertyUtil");
        }

        // 남아 있는 Thread-local Mock 상태 강제 초기화
        Mockito.framework().clearInlineMocks();
    }
}
```

---

#### **테스트 클래스**

```java
import org.junit.jupiter.api.Test;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.annotation.DirtiesContext;
import org.springframework.test.context.TestExecutionListeners;

@SpringBootTest
@DirtiesContext(classMode = DirtiesContext.ClassMode.AFTER_CLASS)
@TestExecutionListeners(
        value = {MockStaticTestExecutionListener.class},
        mergeMode = TestExecutionListeners.MergeMode.MERGE_WITH_DEFAULTS
)
public class ApplicationContextMockTest {

    @Test
    void testApplicationContext() {
        System.out.println("ApplicationContext Loaded!");
        // 테스트 로직
    }
}
```

---

### **6. 해결 방안의 사용 가능성**

1. **Thread-local 상태 강제 초기화**:
   - `Mockito.framework().clearInlineMocks()`는 Thread-local 상태를 명확히 초기화하므로, Mock 중복 생성 문제를 방지합니다.

2. **Mock 생명주기와 테스트 컨텍스트 통합**:
   - `MockStaticTestExecutionListener`를 사용하여 테스트 클래스 수준에서 MockedStatic 상태를 생성, 관리, 정리합니다.

3. **테스트 환경 간 충돌 방지**:
   - `@DirtiesContext`를 통해 ApplicationContext 캐싱을 비활성화하여, 테스트 환경 간 Mock 상태가 충돌하지 않도록 설정합니다.

---

### **7. 결론**

- 본 문제는 MockedStatic 객체의 **Thread-local 상태와 Mock 관리 변수 간의 불일치**에서 발생했습니다.
- 이를 해결하기 위해 `Mockito.framework().clearInlineMocks()`를 사용하여 모든 Mock 상태를 초기화하고, 테스트 생명주기와 Mock 상태를 명확히 관리하도록 설정했습니다.
- 이로써 MockedStatic을 활용한 static 메서드 Mock 처리와 ApplicationContext 초기화 문제를 완전히 해결할 수 있었습니다.

문제가 재발하거나 추가적인 요구사항이 있다면 언제든 말씀해 주세요! 😊
