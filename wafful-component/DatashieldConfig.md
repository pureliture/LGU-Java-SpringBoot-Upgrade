### **DatashieldConfig 클래스 기반 의존성과 동작 방식 분석**

`DatashieldConfig` 클래스는 Datashield 초기화를 담당하며 여러 클래스와 긴밀하게 상호작용합니다. 아래는 의존성과 동작 방식, 역할에 대한 상세한 분석입니다.

---

### **1. 주요 의존성과 빈 주입 관계**
`DatashieldConfig`는 Datashield 설정과 초기화를 위해 여러 클래스를 빈으로 주입하며, 이 클래스들 간의 의존 관계는 다음과 같습니다.

#### 1.1. **`DatashieldInitializer`**
- **역할**:
  - Datashield 서버와 연결하여 키 데이터를 캐싱하고 서버 상태를 초기화합니다.
  - 초기화 중 오류가 발생하더라도, 조건에 따라 애플리케이션 부팅을 계속 진행할 수 있습니다.
- **의존성**:
  - **`EncryptUtil`**: 초기화 성공 여부를 설정합니다.
  - **`DatashieldProperties`**: 설정 속성을 통해 초기화 동작을 제어합니다.
- **주요 메서드**:
  - `testAndCaching()`: Datashield 서버로부터 키 데이터를 캐싱.
  - `monitor()`: 서버 상태를 지속적으로 모니터링.

---

#### 1.2. **`DatashieldDatatypeProperties`**
- **역할**:
  - Datashield에서 관리하는 데이터 타입 및 복합 데이터 타입 정의를 캡슐화합니다.
  - 데이터 타입 관련 설정을 읽고, 중복된 별칭(alias) 여부를 확인하여 예외를 발생시킵니다.
- **주요 메서드**:
  - `throwExceptionIfPrimitiveAliasesIsDuplicated()`: 데이터 타입 별칭 중복 여부 확인.
  - `Builder.build()`: 설정 파일(`datatype-config`)로부터 속성을 초기화.

---

#### 1.3. **`DatashieldApplicationProperties`**
- **역할**:
  - Datashield 애플리케이션 ID와 시스템 코드, 데이터 타입 정보를 관리합니다.
  - 애플리케이션별 암호화/복호화 설정 정보를 제공합니다.
- **주요 메서드**:
  - `getApplicationId(appCode, paramType, cryptoType, datatype)`: 애플리케이션 ID 생성.
  - `Builder.build()`: 설정 파일에서 속성을 읽어 초기화.

---

#### 1.4. **`DatashieldClient`**
- **역할**:
  - Datashield 서버와의 통신을 처리하며, 데이터를 암호화/복호화하거나 상태를 확인합니다.
- **구현체**:
  - `DatashieldClientImpl`: `datashieldDatatype`과 `optionalDatashieldApplicationProperties`를 주입받아 구성됩니다.

---

#### 1.5. **`DeterminedDatatype`**
- **역할**:
  - 특정 데이터 객체와 관련된 Datashield 데이터 타입 및 매핑된 컬럼 정보를 캡슐화합니다.
  - Datashield가 데이터 객체를 처리할 때 데이터 타입과 컬럼 정보를 효율적으로 관리하도록 돕습니다.
- **주요 메서드**:
  - `getDatatype()`: 데이터 타입 반환.
  - `getColumnDescriptorList()`: 컬럼 정보 리스트 반환.
  - `isUnkonw()`: 데이터 타입이 정의되지 않았는지 확인.

---

#### 1.6. **`PropertyDescriptor`의 사용 목적**
- **역할**:
  - 데이터 객체의 속성 메타데이터를 제공하며, Datashield에서 데이터 타입 매핑 및 암호화/복호화 대상 컬럼을 식별하는 데 사용됩니다.
- **주요 기능**:
  - 속성 이름, 타입, Getter/Setter 메서드에 대한 정보를 제공합니다.
  - `DeterminedDatatype`와 `DatashieldDatatypeProperties`에서 컬럼과 데이터 타입의 관계를 설정하는 데 핵심 역할을 합니다.

---

### **2. 동작 방식**
1. **초기화 시점**:
   - Spring Boot 컨텍스트 초기화 시 `DatashieldConfig` 클래스가 로드됩니다.
   - `DatashieldInitializer`는 서버 상태를 확인하고 키 데이터를 캐싱합니다.
   - 데이터 타입과 애플리케이션 속성은 각각 `DatashieldDatatypeProperties`와 `DatashieldApplicationProperties`를 통해 설정됩니다.

2. **런타임 동작**:
   - 클라이언트 요청 시 `DatashieldClient`를 통해 데이터 암호화/복호화 작업이 이루어집니다.
   - `DeterminedDatatype`는 객체의 데이터 타입과 컬럼 정보를 캡슐화하여 암호화 작업을 효율적으로 처리합니다.

---

### **3. 종합적인 클래스 간 역할**
- **`DatashieldConfig`**: Datashield 초기화 및 의존성 관리.
- **`DatashieldInitializer`**: 서버 상태 확인 및 키 데이터 캐싱.
- **`DatashieldProperties`**: 전역 설정 관리.
- **`DatashieldDatatypeProperties`**: 데이터 타입 및 컬럼 매핑 관리.
- **`DeterminedDatatype`**: 특정 데이터 객체의 데이터 타입 및 매핑된 컬럼 정보 관리.
- **`PropertyDescriptor`**: 객체의 속성 메타데이터를 제공하여 데이터 타입 매핑을 지원.

---

### **결론**
이 분석은 `DatashieldConfig`를 중심으로 한 Datashield 설정과 초기화 프로세스에 초점을 맞췄습니다. 각 클래스와의 상호작용 및 `DeterminedDatatype`의 역할을 포함하여 Datashield가 데이터를 처리하는 전체적인 흐름을 이해할 수 있도록 작성되었습니다.

추가 분석이 필요하거나 구체적인 시나리오를 다루고 싶으시면 말씀해 주세요! 😊
