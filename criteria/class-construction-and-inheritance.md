# 클래스 생성과 상속 제한

- 상태: `adopted`
- 주제: `final`, `abstract`, 생성자 접근 제어

## 결론

**생성 경로와 상속 허용 여부를 각각 결정한다.** 상속을 설계한 공통 기반은 `abstract`로 두고 추가 상속이 필요 없는 구체 클래스는 `final`로 닫는다.

**정적 기능만 제공하는 유틸리티 클래스는 `final`과 `private` 생성자를 함께 사용한다.** 외부 생성과 상속을 제한하려는 의도를 선언에 드러낸다.

## 빠른 기준

| 선언 | 해당 클래스의 직접 생성 | 상속 | 선택 조건 |
| --- | --- | --- | --- |
| `final class` | 생성자에 접근할 수 있으면 가능 | 불가 | 완성된 구체 타입의 추가 확장 제한 |
| `abstract class` | 불가 | 생성자에 접근할 수 있으면 가능 | 하위 타입이 공유할 상태와 규칙 제공 |
| 모든 생성자가 `private` | 클래스 내부에서 가능 | 생성자 접근 범위 밖에서는 불가 | 정적 팩토리 등으로 생성 경로 통제 |
| `final`과 `private` 생성자 | 클래스 내부에서 가능 | 불가 | 정적 유틸리티의 외부 생성과 상속 제한 |

`private` 생성자는 같은 최상위 클래스 안의 중첩 코드에서도 호출할 수 있다. 유틸리티 클래스에서는 내부에서도 호출하지 않는 규칙을 둔다. [Java 생성자 접근 규칙](https://docs.oracle.com/javase/specs/jls/se25/html/jls-8.html#jls-8.8.10)

## 유틸리티 예시

```java
public final class NameUtils {
    private NameUtils() {}

    public static String normalize(String name) {
        return name.trim();
    }
}
```

`abstract`는 하위 타입의 공통 기반을 표현할 때 사용한다. 유틸리티에는 `final`과 `private` 생성자를 선택한다. [Java 클래스 선언 규칙](https://docs.oracle.com/javase/specs/jls/se25/html/jls-8.html#jls-8.1.1.1)

객체별 상태나 교체 가능한 동작이 필요하면 인스턴스 객체로 설계한다.

## 관련 기준

- [추상 클래스와 인터페이스 선택](abstract-class-vs-interface.md)
- [필드와 메서드의 final 사용](java-final-keyword.md)
