### `serialVersionUID`의 필요성과 역할

`serialVersionUID`는 **직렬화(Serialization)와 역직렬화(Deserialization)** 과정에서 클래스의 버전을 식별하기 위해 사용되는 **고유한 정수값**입니다. 위 코드에서 `serialVersionUID`가 사용된 이유를 하나씩 설명해볼게요.

---

### 1. `serialVersionUID`가 필요한 이유
자바에서 객체를 파일 또는 네트워크를 통해 전송하려면 **직렬화**가 필요합니다. 직렬화된 객체를 다시 읽어서 사용할 때는 **역직렬화**가 이루어지는데, 이때 **객체의 클래스가 직렬화된 시점과 역직렬화 시점에서 같은지 확인하는 과정**이 필요합니다.

만약 `serialVersionUID`가 없으면, 클래스의 구조가 조금이라도 변경될 경우 **InvalidClassException**이 발생할 수 있습니다.

---

### 2. `serialVersionUID`의 역할
- **클래스 버전 관리**: `serialVersionUID`는 **직렬화된 객체가 호환 가능한지**를 판단하는 기준이 됩니다.
- **자동 생성 시 문제점 방지**: Java는 `serialVersionUID`를 지정하지 않으면 자동으로 생성하는데, 클래스의 필드, 메서드 등이 변경될 때 값이 달라질 가능성이 있습니다.
- **명시적으로 선언하여 안정성 확보**: 직접 `serialVersionUID`를 지정하면 클래스가 변경되더라도 역직렬화가 깨지지 않도록 조정할 수 있습니다.

---

### 3. `serialVersionUID`가 없으면 생기는 문제
다음과 같은 경우에 문제가 발생할 수 있습니다.

#### ✅ 예제: `serialVersionUID` 없이 클래스를 변경한 경우

```java
import java.io.*;

public class Test {
    public static void main(String[] args) throws Exception {
        serialize();
        deserialize(); // 예외 발생 가능
    }

    static void serialize() throws Exception {
        ObjectOutputStream out = new ObjectOutputStream(new FileOutputStream("object.dat"));
        NotAllowedApimRequestException obj = new NotAllowedApimRequestException("Not Allowed");
        out.writeObject(obj);
        out.close();
        System.out.println("Object Serialized!");
    }

    static void deserialize() throws Exception {
        ObjectInputStream in = new ObjectInputStream(new FileInputStream("object.dat"));
        NotAllowedApimRequestException obj = (NotAllowedApimRequestException) in.readObject();
        in.close();
        System.out.println("Object Deserialized: " + obj.getMessage());
    }
}

class NotAllowedApimRequestException extends RuntimeException {
    // serialVersionUID가 없음
    private static final long serialVersionUID = -1728522934603271845L;

    public NotAllowedApimRequestException(String message) {
        super(message);
    }
}
```

이 상태에서 `NotAllowedApimRequestException` 클래스의 구조를 바꾸면(예: 새로운 필드를 추가), 직렬화된 객체를 역직렬화할 때 **`InvalidClassException`** 이 발생할 수 있습니다.

```
Exception in thread "main" java.io.InvalidClassException: 
NotAllowedApimRequestException; local class incompatible: 
stream classdesc serialVersionUID = -1728522934603271845, 
local class serialVersionUID = 123456789L
```

---

### 4. `serialVersionUID`를 지정하는 것이 좋은 이유
- 클래스 변경이 있어도 **호환성을 유지할 수 있음**
- 자동 생성된 `serialVersionUID` 값이 변경되는 문제를 방지
- 직렬화 관련 예외(`InvalidClassException`)를 방지하여 안정성을 높임

---

### 5. `serialVersionUID` 값은 어떻게 생성하나?
직접 선언하는 방법도 있지만, `serialver` 도구를 이용하면 자동으로 값을 생성할 수 있습니다.

```sh
serialver -show
```

```sh
serialver NotAllowedApimRequestException
```

이 명령어를 실행하면 해당 클래스의 `serialVersionUID` 값을 자동으로 생성할 수 있습니다.

---

### 6. 결론
- `serialVersionUID`는 **직렬화된 객체의 클래스 버전을 식별**하는 역할을 합니다.
- **클래스 구조가 변경될 때**도 기존 객체를 올바르게 역직렬화하기 위해 사용됩니다.
- 자동 생성되지 않도록 **직접 값을 지정하는 것이 좋습니다**.
