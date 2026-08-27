# 테스트 전략과 계층별 책임

- 상태: `draft`
- 출처: 기존 테스트 회고와 [방탈출 예약 대기 PR #360](https://github.com/woowacourse/spring-roomescape-waiting/pull/360)의 테스트 파일
- 검토 범위: PR head `356fad7`의 `src/test`만 정적 분석
- 주제: 테스트 목적, 계층별 책임, 통합 테스트, E2E, 불확실성 제어

## 결론

**채택: 테스트는 코드가 실행된다는 사실이 아니라 보호할 규칙을 검증한다.** 계층 이름에 따라 테스트 종류를 하나로 고정하지 않고, 검증하려는 위험에 따라 테스트 경계를 선택한다.

**채택: 도메인은 순수 단위 테스트, Service는 Mock 단위 테스트와 실제 DB 통합 테스트를 구분해 사용한다.** Repository는 쿼리·매핑·DB 제약을 통합 테스트하고, Controller 클래스는 직접 단위 테스트하지 않는다. HTTP 계약은 웹 슬라이스 또는 E2E로 검증한다.

## 빠른 기준

| 대상 | 기본 테스트 | 보호할 규칙 | 피할 테스트 |
| --- | --- | --- | --- |
| 도메인 | 순수 단위 테스트 | 불변식, 경계값, 상태 전이, 권한 규칙 | Spring Context를 띄우거나 협력자를 Mock하는 테스트 |
| Service | Mock 단위 테스트 | 유스케이스 분기, 계산, 협력 순서와 호출 여부 | Mock으로 트랜잭션 성공·롤백을 증명하는 테스트 |
| Service 트랜잭션 | 실제 DB 통합 테스트 | 여러 Repository 협력, 트랜잭션 원자성, 롤백 | 성공 경로만 확인하고 중간 실패를 만들지 않는 테스트 |
| Repository | `@JdbcTest` 등 DB 통합 테스트 | SQL, Row Mapping, 정렬, 제약조건, 삭제 영향 | Repository를 Mock하고 쿼리가 맞다고 판단하는 테스트 |
| Controller | 직접 단위 테스트하지 않음 | 직접 보호할 도메인 규칙 없음 | Service 위임만 검증하는 Controller Mock 테스트 |
| HTTP API | 웹 슬라이스 또는 E2E | 요청 바인딩, 검증, 상태 코드, 오류 형식, 직렬화 | 모든 도메인 경계값을 E2E에서 반복하는 테스트 |
| 애플리케이션 기동 | 선택적 Smoke Test | Spring Bean 구성과 Context 기동 | 동작 검증을 `contextLoads()`로 대체하는 테스트 |
| 시간·랜덤·외부 API | 제어 가능한 의존성 주입 | 같은 입력에 대한 결정적 결과 | 시스템 시각·실제 네트워크·실제 난수에 의존하는 테스트 |

## 1. PR 테스트 파일에서 확인한 구조

PR head의 테스트 파일만 확인했다. 운영 코드와 PR 본문은 판단 근거에서 제외했다.

| 구분 | 파일 수 | 테스트 수 | 확인한 도구와 경계 |
| --- | ---: | ---: | --- |
| 도메인 단위 테스트 | 4 | 17 | JUnit·AssertJ, Spring 미사용 |
| Service Mock 단위 테스트 | 4 | 21 | Mockito, Repository·Query Service 대역 |
| Repository 통합 테스트 | 4 | 34 | `@JdbcTest`, 실제 JDBC Repository, H2 |
| API E2E | 4 | 65 | `@SpringBootTest(DEFINED_PORT)`, RestAssured |
| Context Smoke Test | 1 | 1 | 빈 `contextLoads()` |

테스트 패키지는 `unit/domain`, `unit/service`, `integration`, `e2e`로 구분돼 있다. 이름만으로 테스트 경계가 드러난다는 점은 **유지**한다.

### 확인하지 못한 내용

- 테스트 실행 결과와 실행 시간
- 운영 DB 종류와 H2의 호환성
- Service의 실제 트랜잭션 선언과 전파 속성
- Controller가 실제로 얇게 유지되는지 여부

위 항목은 운영 코드를 보지 않았으므로 **확인 필요**다.

## 2. 테스트 대상은 구현이 아니라 규칙

초기에는 `doesNotThrowAnyException()`으로 생성 또는 실행 성공만 확인했지만, 테스트의 목적을 다음 질문으로 바꾼다.

> 이 테스트가 실패하면 어떤 규칙이 깨졌다는 뜻인가?

명확한 답이 없다면 테스트를 만들지 않거나 더 구체적인 결과를 검증한다.

### `doesNotThrowAnyException()` 사용 기준

`doesNotThrowAnyException()` 자체를 일괄적으로 거부하지 않는다.

- 채택: 허용되는 정확한 경계값을 보호할 때
- 채택: 반환값이 없는 명령의 성공이 도메인 규칙일 때
- 거부: 기본 생성이나 단순 메서드 호출이 실행된다는 사실만 확인할 때
- 권장: 가능하면 생성된 상태나 발생한 협력 결과도 함께 확인

PR의 테스트에서는 다음처럼 허용 경계를 검증하고 있다.

- 이름은 10자까지 허용하고 11자부터 거부한다.
- 미래 예약은 허용하고 지난 예약은 거부한다.
- 본인의 미래 예약 취소는 허용하고 타인 또는 지난 예약 취소는 거부한다.

이 경우 예외가 발생하지 않는다는 사실은 반대편 실패 사례와 함께 경계 규칙을 보호하므로 **채택**한다.

## 3. 도메인: 순수 단위 테스트

도메인 테스트는 Spring 없이 객체를 직접 생성해 실행한다.

검증 대상:

- 생성 불변식
- 유효값과 무효값의 경계
- 상태 전이
- 소유자·권한 규칙
- 시간에 따른 도메인 판단
- 같은 입력에 대한 결정적 결과

PR의 `MemberTest`, `ReservationTest`, `ReservationWaitingTest`, `SlotTest`는 이 경계를 따른다. 날짜와 현재 시각을 고정값으로 전달하고, 도메인 예외와 결과 상태를 검증한다.

**거부: 도메인 객체를 테스트하기 위해 Spring Context를 띄우거나 Repository를 Mock하지 않는다.** 도메인이 인프라 없이 테스트되지 않는다면 의존 방향을 다시 검토한다.

## 4. Service: 두 종류의 테스트

Service는 하나의 테스트 방식으로 충분하지 않다.

### 4.1 Mock 단위 테스트

Mock 단위 테스트는 Service가 내리는 결정과 협력을 빠르게 검증한다.

검증 대상:

- 조회 결과에 따른 성공·실패 분기
- Repository 호출 또는 호출 방지
- 페이지 offset, 조회 기간 같은 계산
- 여러 도메인 객체와 협력자를 조합하는 유스케이스
- 도메인 규칙 위반 시 이후 저장을 실행하지 않는지 여부

PR의 Service 테스트는 `MockitoExtension`, `given`, `verify`, `never`를 사용한다. 다음 규칙을 확인한다.

- 존재하지 않는 대상은 `NotFoundException`을 발생시킨다.
- 본인의 미래 예약만 삭제 Repository에 위임한다.
- 예약되지 않은 슬롯에는 대기를 저장하지 않는다.
- 페이지와 크기로 offset을 계산한다.
- 인기 테마 조회 기간을 계산해 Repository에 전달한다.

이 테스트는 비즈니스 유스케이스의 분기와 협력 계약을 보호한다. 빠르고 실패 원인이 좁다는 장점이 있다.

### 4.2 실제 DB 통합 테스트

Mock은 설정한 대로만 동작하므로 다음 항목을 증명할 수 없다.

- 실제 트랜잭션 경계
- 두 번 이상의 DB 변경이 하나의 원자적 작업인지 여부
- 중간 실패 시 전체 변경이 롤백되는지 여부
- 실제 제약조건과 예외 변환
- 트랜잭션 전파와 잠금 동작

여러 Repository가 하나의 유스케이스에서 데이터를 변경한다면 실제 DB를 사용하는 Service 통합 테스트를 추가한다.

검증 방법:

1. 테스트 DB에 초기 상태를 저장한다.
2. 첫 번째 DB 변경 이후 실패하도록 제어 가능한 협력자를 구성한다.
3. Service 유스케이스를 실행해 예외를 발생시킨다.
4. 모든 Repository를 다시 조회해 첫 번째 변경도 롤백됐는지 확인한다.

PR의 테스트 트리에는 Service와 실제 Repository를 함께 사용하는 전용 통합 테스트와 인위적 실패를 통한 롤백 검증이 없다. 따라서 트랜잭션 원자성은 **확인 필요**다.

## 5. Repository: 쿼리와 매핑 통합 테스트

Repository 테스트는 메서드가 호출됐는지가 아니라 실제 DB 결과를 검증한다.

검증 대상:

- 저장 후 생성 ID와 조회 결과
- SQL 조건과 정렬 순서
- Row Mapping
- Unique·Foreign Key 같은 DB 제약
- 삭제와 연관 데이터 처리
- 집계, 기간 조건, limit
- 존재하지 않는 데이터의 반환 규칙

PR의 네 Repository 테스트는 `@JdbcTest`와 실제 JDBC Repository를 사용한다. 생성 ID, 중복 예약, 조회 조건, 정렬, 삭제, 대기 순번, 인기 테마 기간 집계를 검증한다. 이 방향은 **채택**한다.

### DB 선택 제약

테스트 설정은 H2 인메모리 DB를 사용한다. SQL 문법, 잠금, 트랜잭션 격리, 제약조건 동작이 운영 DB에 의존한다면 H2만으로 충분하지 않다.

- DB 비종속 CRUD·매핑: H2 기반 `@JdbcTest` 유지 가능
- 운영 DB 전용 SQL·잠금·동시성: Testcontainers로 같은 DB 엔진 사용
- 선택 기준: 테스트 속도보다 운영 DB와의 의미 차이가 장애 위험을 만드는지 여부

운영 DB 종류는 테스트 파일만으로 확인하지 못했으므로 Testcontainers 필요 여부는 **보류**한다.

## 6. Controller와 HTTP API

Controller 클래스의 메서드를 직접 호출하고 Service 위임을 검증하는 단위 테스트는 기본적으로 만들지 않는다.

- Controller가 단순 입력 변환과 Service 호출만 담당한다면 검증 가치가 낮다.
- Controller에 비즈니스 분기가 있다면 테스트를 추가하기보다 해당 규칙을 Service 또는 도메인으로 이동한다.
- 요청 바인딩, Bean Validation, 예외 매핑은 Java 메서드 직접 호출로 검증할 수 없다.

PR 테스트에는 `@WebMvcTest`나 MockMvc 기반 Controller 테스트가 없다. 대신 실제 포트에서 RestAssured로 HTTP 요청을 보내는 E2E 테스트가 다음을 검증한다.

- HTTP 상태 코드
- JSON 요청·응답
- 필수값과 형식 검증
- 오류 응답 형식
- 생성 후 조회·변경·삭제 흐름
- 실제 Repository까지 연결된 결과

**채택: Controller 직접 단위 테스트 대신 외부에 공개한 HTTP 계약을 검증한다.**

### 웹 슬라이스를 선택하는 경우

다음 요구가 많고 E2E 비용이 커지면 `@WebMvcTest`를 사용한다.

- 요청 파라미터와 JSON 바인딩 조합
- Bean Validation 메시지
- `ControllerAdvice` 오류 매핑
- 인증·인가 필터
- 콘텐츠 타입과 직렬화 형식

웹 슬라이스는 HTTP 계층만 빠르게 검증한다. DB 저장이나 트랜잭션 성공까지 증명하지는 않는다.

## 7. E2E: 핵심 계약만 검증

E2E는 사용자 관점에서 전체 시스템 연결을 검증한다.

우선 포함할 시나리오:

- 주요 API의 대표 성공 흐름
- 요청 검증과 오류 형식의 대표 사례
- 생성 후 조회처럼 여러 요청이 이어지는 핵심 흐름
- 여러 계층 연결에서만 발견할 수 있는 직렬화·매핑 문제
- 가장 중요한 권한과 트랜잭션 결과

낮은 계층으로 내릴 시나리오:

- 문자열 길이와 날짜 경계의 모든 조합
- 같은 도메인 예외의 반복
- Repository 정렬·집계 SQL의 세부 경우
- Service의 단순 분기와 호출 여부

### 현재 PR에 대한 판단

API E2E는 4개 파일에 65개로, 도메인과 Service 단위 테스트 38개보다 많다. HTTP 계약뿐 아니라 도메인·Repository의 세부 경계도 반복해서 검증한다.

**판단: E2E 비중을 줄인다.** 하위 테스트에서 원인을 빠르게 확인할 수 있는 규칙은 도메인·Service·Repository 테스트로 내리고, E2E는 계층 연결과 대표 계약에 집중한다.

네 E2E 클래스는 모든 테스트 메서드 전에 `@DirtiesContext`로 Application Context를 다시 만든다. 격리는 강하지만 Context 재기동 비용이 크다.

대안:

- 테스트별 명시적 DB 초기화
- 독립 스키마 또는 컨테이너 재사용 전략
- E2E 시나리오 수 축소
- 데이터 생성 Fixture와 정리 전략 통일

실제 포트로 요청하는 E2E에서는 테스트 메서드의 `@Transactional` 롤백이 서버 요청의 트랜잭션까지 감싸지 못할 수 있다. 단순히 `@Transactional`을 추가하는 방식은 대안으로 채택하지 않는다.

## 8. 시간·랜덤·외부 API 제어

불확실한 요소는 테스트가 제어할 수 있는 의존성으로 바꾼다.

### 시간

- 애플리케이션 경계에서 `Clock` 주입
- 테스트에서는 `Clock.fixed(...)` 사용
- 도메인에는 계산된 현재 시각을 명시적으로 전달
- `LocalDate.now()`와 `LocalDateTime.now()` 직접 호출 지양

PR의 `ReservationUseCaseMockTest`와 `ReservationWaitingUseCaseMockTest`는 고정 `Clock`을 사용한다. 도메인 테스트는 현재 시각을 매개변수로 전달한다. 이 구조는 **채택**한다.

### 랜덤

- `RandomGenerator` 같은 계약으로 분리
- 테스트에서는 고정값 또는 순서가 정해진 Stub 사용
- Java 라이브러리의 `shuffle()` 알고리즘 자체는 다시 검증하지 않음
- 검증 대상은 셔플 사용 여부가 아니라 애플리케이션이 랜덤 결과를 처리하는 규칙

### 외부 API

- HTTP Client를 포트로 분리
- 단위 테스트에서는 Fake·Stub 사용
- 요청·응답 매핑은 WireMock·MockWebServer 같은 경계 통합 테스트 사용
- 실제 외부 서비스 호출은 기본 테스트 경로에서 제외

PR 테스트에는 랜덤과 외부 API 사례가 없어 해당 기준의 적용 결과는 **확인 필요**다.

## 9. `contextLoads()` 판단

빈 `contextLoads()`는 애플리케이션 Context가 구성된다는 한 가지 사실만 보호한다.

- 별도 Smoke Test 단계가 필요하면 유지 가능
- 이미 모든 E2E가 동일한 `@SpringBootTest` Context를 기동한다면 중복 가치가 낮음
- 도메인·Service·API 동작이 정상이라는 근거로 사용할 수 없음

현재 PR에서는 65개 E2E가 실제 Context를 기동하므로 별도 `contextLoads()` 테스트는 **거부**한다. CI에서 최소 기동 확인만 별도 실행하려는 목적이 있다면 유지 여부를 다시 판단한다.

## 최종 판단 기준

> 테스트는 실행 여부가 아니라 깨지면 안 되는 규칙을 보호한다. 도메인은 순수 단위 테스트로 불변식과 상태 전이를 검증한다. Service는 Mock 단위 테스트로 유스케이스 분기를 검증하고, 여러 Repository와 트랜잭션이 협력하면 실제 DB 통합 테스트로 원자성과 롤백을 검증한다. Repository는 실제 쿼리와 매핑을 검증한다. Controller는 직접 단위 테스트하지 않고 HTTP 계약을 웹 슬라이스 또는 소수의 E2E로 검증한다. 시간·랜덤·외부 API는 제어 가능한 의존성으로 바꾼다.

## 근거

- [방탈출 예약 대기 PR #360](https://github.com/woowacourse/spring-roomescape-waiting/pull/360)
- [분석한 테스트 디렉터리](https://github.com/kangrae-jo/spring-roomescape-waiting/tree/356fad7de9f3a1bf40e3819e314c51a6245fd36e/src/test)
- [도메인 단위 테스트](https://github.com/kangrae-jo/spring-roomescape-waiting/tree/356fad7de9f3a1bf40e3819e314c51a6245fd36e/src/test/java/roomescape/unit/domain)
- [Service Mock 단위 테스트](https://github.com/kangrae-jo/spring-roomescape-waiting/tree/356fad7de9f3a1bf40e3819e314c51a6245fd36e/src/test/java/roomescape/unit/service)
- [Repository 통합 테스트](https://github.com/kangrae-jo/spring-roomescape-waiting/tree/356fad7de9f3a1bf40e3819e314c51a6245fd36e/src/test/java/roomescape/integration)
- [API E2E](https://github.com/kangrae-jo/spring-roomescape-waiting/tree/356fad7de9f3a1bf40e3819e314c51a6245fd36e/src/test/java/roomescape/e2e)

## 후속 검증

- [ ] 실제 DB를 사용해 여러 Repository 변경의 롤백 테스트를 추가한 사례를 연결한다.
- [ ] E2E 축소 전후 실행 시간과 결함 탐지 범위를 비교한다.
- [ ] 운영 DB가 H2와 다르면 Repository 핵심 쿼리를 동일 DB 엔진에서 재검증한다.
