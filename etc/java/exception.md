# Java 예외(Exception) 이해와 활용 전략

이 문서는 Java의 예외 계층 구조를 이해하고, 체크 예외와 언체크 예외의 특징과 올바른 활용 전략에 대해 설명합니다. 특히 계층형 아키텍처에서 예외 처리가 의존 관계에 미치는 영향과 이를 해결하기 위한 방법을 중점적으로 다룹니다.

## 1\. Java 예외 계층 구조
<img width="811" height="494" alt="스크린샷 2025-09-15 오후 5 50 16" src="https://github.com/user-attachments/assets/0443589c-8012-4202-ae3d-c22369da4490" />

Java의 모든 예외 클래스는 `Object` 클래스를 상속받으며, 주요 계층은 다음과 같습니다. 

  - **`Object`**: 모든 자바 객체의 최상위 클래스입니다.
  - **`Throwable`**: 예외 계층의 최상위 클래스로, `Exception`과 `Error`로 나뉩니다. 
      - **`Error`**: 시스템 레벨의 심각한 오류로, 애플리케이션에서 복구할 수 없는 상황을 나타냅니다 (예: `OutOfMemoryError`).개발자는 `Error`를 잡으려고 시도해서는 안 됩니다.
      - **`Exception`**: 애플리케이션 로직에서 처리 가능한 예외의 최상위 클래스입니다.
          - **체크 예외 (Checked Exception)**: `RuntimeException`을 제외한 `Exception`의 모든 하위 클래스입니다. 컴파일러가 처리를 강제합니다. 
          - **언체크 예외 (Unchecked Exception)**: `RuntimeException`과 그 하위 클래스들을 의미합니다. 컴파일러가 처리를 강제하지 않습니다.

## 2\. 예외 처리 기본 규칙
<img width="814" height="603" alt="스크린샷 2025-09-15 오후 5 50 49" src="https://github.com/user-attachments/assets/52769881-2faf-4e73-85cc-2a84f7f68ea3" />

1. **예외는 잡아서 처리하거나(Catch), 밖으로 던져야(Throw) 합니다.** 

      - **예외 처리**: `try-catch` 구문을 사용해 예외를 잡고, 적절한 로직을 수행하면 프로그램 흐름이 정상적으로 돌아옵니다.
      - **예외 던지기**: 처리할 수 없는 예외는 `throws` 키워드를 통해 호출한 메서드로 전파합니다. 만약 `main()` 메서드까지 예외가 던져지면 프로그램은 종료됩니다.웹 애플리케이션(WAS)에서는 예외를 받아 오류 페이지를 사용자에게 보여줍니다. 

2. **예외를 잡거나 던질 때, 해당 예외와 그 자식 예외들이 함께 처리됩니다.** 

      - 예를 들어, `catch (Exception e)`는 `Exception`의 모든 자식 예외(예: `IOException`, `SQLException`)를 잡을 수 있습니다.

## 3\. 체크 예외 (Checked Exception)

`Exception` 클래스를 상속하고 `RuntimeException`이 아닌 예외입니다.

  - **특징**: 반드시 `try-catch`로 처리하거나, `throws`로 던져야 하며, 그렇지 않으면 컴파일 오류가 발생합니다.
  - **장점**: 컴파일러가 예외 처리를 강제하므로 개발자가 예외를 누락하는 실수를 방지할 수 있습니다.
  - **단점**: 모든 예외를 강제로 처리해야 하므로 코드가 번거로워지고, 의존 관계에 문제를 일으킬 수 있습니다. 

### 체크 예외의 문제점

체크 예외는 컴파일러를 통해 예외 처리를 강제하는 장점이 있지만, 실제 애플리케이션에서는 다음과 같은 두 가지 큰 문제점을 가집니다.

1.  **복구 불가능한 예외**
    * `SQLException`과 같이 데이터베이스나 네트워크 통신 등 시스템 레벨에서 발생하는 예외는 대부분 애플리케이션 비즈니스 로직에서 의미 있는 복구가 불가능합니다. 예를 들어, SQL 문법 오류나 데이터베이스 서버 다운과 같은 문제는 서비스나 컨트롤러 계층에서 해결할 수 없습니다.
    * 이러한 예외들은 결국 처리되지 못하고 계속 상위로 던져지게 되며, 최종적으로는 공통 예외 처리 로직(예: 서블릿 필터, 스프링의 `ControllerAdvice`)에서 일관되게 처리해야 합니다.

2.  **의존 관계 문제**
    * Controller → Service → Repository 구조에서 Repository가 `SQLException`(체크 예외)을 던진다고 가정해 보겠습니다.
    * Service는 `SQLException`을 복구할 수 없으므로, `throws SQLException`을 메서드에 선언하여 Controller에게 던져야 합니다.
    * Controller 역시 처리할 수 없으므로, `throws SQLException`을 선언해 밖으로 던져야 합니다.
    * **문제**: 이로 인해 Service와 Controller가 JDBC 기술에 특화된 `java.sql.SQLException`에 직접 의존하게 됩니다. 서비스나 컨트롤러 입장에서는 본인이 처리할 수도 없는 예외를 의존해야 하는 큰 단점이 발생합니다.
    * **파급 효과**: 만약 Repository의 기술을 JDBC에서 JPA로 변경하여 `JPAException`이 발생하게 되면, `SQLException`에 의존하던 모든 Service와 Controller의 코드를 `JPAException`으로 수정해야 하는 연쇄적인 변경이 발생합니다. 이는 구현 기술의 변경이 상위 계층까지 영향을 미치는 상황으로, OCP(개방-폐쇄 원칙)를 위반하게 됩니다.
<!-- end list -->

```java
// Service와 Controller가 SQLException에 의존하는 문제 코드
class Service {
    public void logic() throws SQLException, ConnectException { // 구체적인 예외에 의존
        repository.call();
        networkClient.call();
    }
}

class Controller {
    public void request() throws SQLException, ConnectException { // 상위 계층까지 의존 관계 전파
        service.logic();
    }
}
```

## 4\. 언체크 예외 (Unchecked Exception)

<img width="825" height="303" alt="스크린샷 2025-09-15 오후 6 11 03" src="https://github.com/user-attachments/assets/ea1bc0f6-d18b-46e7-82ae-7b1085e6cf89" />

`RuntimeException` 클래스를 상속받은 예외입니다. 

  - **특징**: `throws`를 선언하지 않아도 되며, 예외를 잡지 않으면 자동으로 밖으로 던져집니다.
  - **장점**: 처리할 수 없는 예외를 무시할 수 있어 코드가 간결해지고, 불필요한 의존 관계를 만들지 않습니다.
  - **단점**: 컴파일러가 확인해주지 않아 개발자가 실수로 예외 처리를 누락할 수 있습니다.따라서 **문서화**가 매우 중요합니다. 

### 언체크 예외 활용 전략: 예외 전환 (Exception Translation)

체크 예외가 발생하는 가장 최하위 계층(예: Repository)에서 이를 언체크 예외로 전환하여 던지는 것이 효과적입니다.

1.  **체크 예외를 언체크 예외로 감싸서 던집니다.**
      - Repository에서 `SQLException`이 발생하면, 이를 `RuntimeSQLException`과 같은 언체크 예외로 전환하여 던집니다.
2.  **상위 계층의 의존성을 제거합니다.**
      - Service와 Controller는 더 이상 `SQLException`에 의존할 필요가 없으므로 `throws` 구문을 작성하지 않아도 됩니다.
3.  **유지보수성이 향상됩니다.**
      - 향후 Repository 기술이 변경되어도 Service나 Controller 코드는 수정할 필요가 없습니다.변경의 영향 범위가 최소화됩니다.

<!-- end list -->

```java
// Repository에서 예외를 전환하는 코드
class Repository {
    public void call() {
        try {
            runSQL();
        } catch (SQLException e) {
            // 체크 예외를 언체크 예외로 전환
            // 반드시 기존 예외(e)를 포함해야 스택 트레이스 추적이 용이하다.
            throw new RuntimeSQLException(e);
        }
    }
    private void runSQL() throws SQLException {
        throw new SQLException("ex");
    }
}

// Service와 Controller는 더 이상 특정 예외에 의존하지 않는다.
class Service {
    public void logic() {
        repository.call();
        networkClient.call();
    }
}
```

## 5\. 예외 전환 시 핵심: 기존 예외 포함

예외를 전환할 때는 반드시 `new RuntimeSQLException(e)`와 같이 생성자에 기존 예외(`cause`)를 넘겨주어야 합니다.

  - **이유**: 이렇게 해야 로그에 `Caused by:` 부분이 포함되어, 최초에 어떤 근본 원인으로 예외가 발생했는지 스택 트레이스를 통해 명확히 확인할 수 있습니다. 
  - 만약 기존 예외를 포함하지 않으면, 최초 발생한 `SQLException`의 정보를 잃게 되어 디버깅이 매우 어려워지는 심각한 문제가 발생합니다. 
## 6\. 결론 및 기본 원칙

  - **기본적으로 언체크(런타임) 예외를 사용합니다.**
      - 대부분의 예외는 복구가 불가능하며, 이런 예외는 공통으로 처리하는 것이 효율적입니다. 
  - **체크 예외는 비즈니스 로직상 반드시 처리해야 하는 경우에만 사용합니다.** 
      - 예: 계좌 이체 실패, 포인트 부족 등 호출한 쪽에서 명확하게 인지하고 복구 로직을 수행해야 할 때 유용합니다. 
  - **런타임 예외는 문서화를 통해 명확히 알리거나, 중요한 경우 `throws`를 명시해 IDE의 도움을 받을 수 있습니다.** 
