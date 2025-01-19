# Java `PropertyDescriptor`를 활용한 클래스 멤버 필드 접근 및 데이터 패턴 변경

## 개요
Java의 `PropertyDescriptor`는 리플렉션 기반의 API로, 클래스의 속성(멤버 변수)에 쉽게 접근하고 값을 읽거나 설정할 수 있는 기능을 제공합니다. 본 포스팅에서는 **특정 클래스 타입의 모든 멤버 필드를 읽어 들인 뒤 특정 패턴에 매칭되는 데이터를 변경**하는 기능을 구현하는 방법을 설명합니다.

---

## 주요 목표
- 클래스 타입(`Class<T>`)을 받아 멤버 필드 정보를 동적으로 읽어들임.
- 특정 패턴(예: 정규 표현식)에 매칭되는 필드 값을 탐색.
- 매칭된 데이터를 원하는 값으로 변경.

---

## 주요 기술 및 도구
- **`PropertyDescriptor`**: 자바빈(JavaBean)의 속성을 읽고 쓸 수 있도록 지원.
- **`java.lang.reflect`**: 리플렉션을 통해 클래스 정보에 동적으로 접근.
- **정규 표현식**: 데이터의 패턴 매칭 및 필터링.

---

## 구현 전략

### 1. 기본 클래스 설계
테스트를 위해 사용할 샘플 클래스를 설계합니다.

```java
public class User {
    private String name;
    private String email;
    private int age;

    // Getters and Setters
    public String getName() {
        return name;
    }
    public void setName(String name) {
        this.name = name;
    }
    public String getEmail() {
        return email;
    }
    public void setEmail(String email) {
        this.email = email;
    }
    public int getAge() {
        return age;
    }
    public void setAge(int age) {
        this.age = age;
    }
}
```

### 2. `PropertyDescriptor` 활용 기능 구현
클래스의 멤버 필드를 동적으로 접근하여 특정 패턴을 찾고 값을 변경합니다.

```java
import java.beans.PropertyDescriptor;
import java.lang.reflect.Field;
import java.lang.reflect.Method;
import java.util.regex.Pattern;

public class PropertyModifier {

    /**
     * 특정 클래스 인스턴스에서 패턴이 매칭되는 필드 값을 변경
     * 
     * @param target      데이터가 담긴 클래스 인스턴스
     * @param fieldType   처리할 필드 타입 (예: String.class)
     * @param pattern     매칭할 정규 표현식
     * @param replacement 변경할 값
     * @param <T>         클래스 타입
     * @throws Exception 리플렉션 관련 예외
     */
    public static <T> void modifyFields(T target, Class<?> fieldType, String pattern, String replacement) throws Exception {
        Pattern regexPattern = Pattern.compile(pattern);

        // 모든 필드에 대해 순회
        for (Field field : target.getClass().getDeclaredFields()) {
            // 필드 타입 체크
            if (field.getType().equals(fieldType)) {
                PropertyDescriptor pd = new PropertyDescriptor(field.getName(), target.getClass());
                Method getter = pd.getReadMethod(); // Getter
                Method setter = pd.getWriteMethod(); // Setter

                // 필드 값 읽기
                Object fieldValue = getter.invoke(target);

                // 패턴 매칭 및 값 변경
                if (fieldValue instanceof String && regexPattern.matcher((String) fieldValue).find()) {
                    setter.invoke(target, replacement);
                }
            }
        }
    }
}
```

### 3. 테스트 코드
위에서 구현한 `PropertyModifier`를 활용하여 특정 필드 값을 변경하는 테스트 코드를 작성합니다.

```java
public class PropertyModifierTest {

    public static void main(String[] args) throws Exception {
        // 테스트 객체 생성
        User user = new User();
        user.setName("Alice");
        user.setEmail("alice@example.com");
        user.setAge(30);

        // 변경 전 상태 출력
        System.out.println("Before:");
        System.out.println("Name: " + user.getName());
        System.out.println("Email: " + user.getEmail());
        System.out.println("Age: " + user.getAge());

        // 특정 패턴 매칭 후 데이터 변경
        PropertyModifier.modifyFields(user, String.class, "example\\.com", "updated@example.com");

        // 변경 후 상태 출력
        System.out.println("\nAfter:");
        System.out.println("Name: " + user.getName());
        System.out.println("Email: " + user.getEmail());
        System.out.println("Age: " + user.getAge());
    }
}
```

---

## 실행 결과
위 코드를 실행하면 아래와 같은 결과를 얻을 수 있습니다.

### 실행 전 출력
```
Before:
Name: Alice
Email: alice@example.com
Age: 30
```

### 실행 후 출력
```
After:
Name: Alice
Email: updated@example.com
Age: 30
```

---

## 심화: 추가 개선 방안
1. **필드 타입 필터링**:
   - 특정 타입(String, Integer 등)만 필터링하도록 확장.
2. **재귀 지원**:
   - 중첩된 객체의 필드도 접근 가능하도록 개선.
3. **컬렉션 처리**:
   - `List`나 `Map` 같은 컬렉션 내부 값도 매칭 및 변경.
