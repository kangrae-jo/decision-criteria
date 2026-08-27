# 검증 책임과 DB 제약

- 상태: `validated`
- 출처: [방탈출 사용자 예약 PR #451](https://github.com/woowacourse/spring-roomescape-member/pull/451)의 구현과 코드 리뷰
- 검토 기준: PR 최종 커밋 `1792fad`
- 주제: 입력 검증, 비즈니스 유효성, DB 무결성, 중복 검사

## 결론

**채택: 검증을 입력 형식, 비즈니스 의미, 영속 데이터 무결성으로 분리한다.** 각 계층은 같은 값을 보더라도 서로 다른 실패를 방지한다.

**채택: 동시 요청에서 반드시 지켜야 하는 규칙은 DB 제약조건으로 보장한다.** Service의 사전 조회는 사용자에게 더 구체적인 안내가 필요할 때만 추가한다.

**거부: 모든 DB 제약을 Service에서 동일하게 사전 검사하지 않는다.** 조회 후 저장 사이의 경쟁 조건 때문에 Service 검사만으로 무결성을 보장할 수 없고, 불필요한 쿼리와 중복 로직이 늘어난다.

## 빠른 기준

| 검증 종류 | 책임 위치 | 예시 | 실패 목적 |
| --- | --- | --- | --- |
| 요청 형식·구조 | Payload·웹 경계 | 필수값, JSON 형식, 날짜 파싱, 문자열 길이 | 처리할 수 없는 요청 조기 거부 |
| 단일 객체 불변식 | 도메인 | 이름 규칙, 상태 전이 가능 여부 | 모든 생성 경로에서 유효한 객체 보장 |
| 유스케이스 유효성 | Service | 자원 존재, 소유권, 과거 예약, 외부 상태 | 비즈니스 의미에 맞는 판단과 안내 |
| 영속 데이터 무결성 | DB 제약 | `UNIQUE`, `FOREIGN KEY`, `NOT NULL`, `CHECK` | 동시 요청과 우회 경로에서도 최종 상태 보호 |
| DB 오류 해석 | Repository Adapter | `DuplicateKeyException` 변환 | 기술 예외를 안정적인 애플리케이션 오류로 변환 |

## 1. 같은 규칙을 여러 계층에서 확인할 수 있다

검증 코드가 여러 계층에 있다는 사실만으로 중복이라고 판단하지 않는다. 먼저 목적이 같은지 확인한다.

```text
Payload
→ 요청을 읽을 수 있는가?

도메인·Service
→ 현재 맥락에서 허용되는가?

DB
→ 어떤 경로와 동시성 상황에서도 저장 상태가 유효한가?
```

예를 들어 중복 예약은 Service와 DB에서 모두 다룰 수 있지만 목적이 다르다.

- Service 사전 조회: 사용자가 다음 행동을 결정할 수 있는 상세 안내
- DB 유일성 제약: 두 요청이 동시에 들어와도 중복 저장 방지

이 둘은 대체 관계가 아니다. Service 검사를 추가해도 DB 제약은 유지한다.

## 2. Payload: 입력 형식과 API 계약

Payload는 요청을 애플리케이션이 안전하게 해석할 수 있는지 검증한다.

검증 대상:

- 필수 필드 누락
- 빈 문자열
- JSON 문법과 타입
- 날짜·시간 형식
- API 계약에 명시된 문자열·숫자 범위
- 컬렉션 크기

PR의 `ReservationRequest`는 다음 항목을 Bean Validation으로 확인한다.

- 예약자 이름 `@NotBlank`
- 예약자 이름 최대 10자
- 날짜, 시간 ID, 테마 ID `@NotNull`

Payload 검증 실패는 일반적으로 `400 Bad Request`로 반환한다.

### 제약

Payload 검증은 HTTP 요청 경로만 보호한다. 내부 배치, 메시지 소비자, 테스트 Fixture, 다른 Service가 도메인 객체를 직접 생성할 수 있다.

따라서 이름 길이처럼 객체가 항상 지켜야 하는 규칙이라면 Payload에만 두지 않는다.

- API 입력 계약: Payload
- 모든 생성 경로에서 지킬 불변식: 도메인 생성자 또는 Factory

**판단: 빠른 요청 피드백을 위해 Payload에서 검증할 수 있지만, 도메인 불변식의 유일한 방어선으로 사용하지 않는다.**

## 3. 도메인과 Service: 비즈니스 의미

형식이 올바른 값도 현재 비즈니스 맥락에서는 허용되지 않을 수 있다.

### 도메인이 판단할 규칙

- 객체 내부 상태만으로 판단 가능
- 모든 호출 경로에서 동일하게 적용
- 상태 전이와 불변식에 해당

예시:

- 예약 소유자인가?
- 현재 상태에서 취소할 수 있는가?
- 이름이 도메인 규칙에 맞는가?

### Service가 판단할 규칙

- 여러 객체나 Repository가 필요
- 현재 시각이나 사용자 정보가 필요
- 외부 API 또는 다른 도메인의 결과가 필요
- 어떤 실패인지 구분해 사용자의 다음 행동을 안내해야 함

PR의 `ReservationService`는 다음을 판단한다.

- 예약 시간과 테마가 존재하는가?
- 요청한 예약 시각이 과거인가?
- 변경·취소 요청자가 예약 소유자인가?
- 대상 예약이 존재하는가?

이 규칙들은 요청 형식이나 테이블 구조만으로 의미를 결정할 수 없으므로 Service 또는 도메인 책임이다.

## 4. DB: 최종 무결성

DB 제약은 애플리케이션의 모든 저장 경로와 동시 요청에서 깨지면 안 되는 상태를 보호한다.

PR의 `reservation` 테이블은 다음 제약을 가진다.

```sql
UNIQUE (date, theme_id, time_id)
FOREIGN KEY (time_id) REFERENCES reservation_time (id)
FOREIGN KEY (theme_id) REFERENCES theme (id)
```

`UNIQUE (date, theme_id, time_id)`는 같은 날짜·테마·시간의 예약을 하나만 허용한다.

### DB 제약으로 둘 규칙

- 기본 키
- 필수 컬럼의 `NOT NULL`
- 참조 무결성
- 어떤 저장 경로에서도 허용되지 않는 중복
- 값 범위가 데이터 자체의 영구 불변식인 `CHECK`

### DB 제약으로 두지 않을 규칙

- 현재 시각보다 과거인지 여부
- 사용자 등급에 따라 달라지는 정책
- 관리자와 일반 사용자에게 다르게 적용되는 규칙
- 외부 API 결과에 따라 바뀌는 판단
- 운영 중 자주 변경될 비즈니스 정책

예를 들어 과거 예약 생성 금지는 현재 시각과 사용자 역할에 따라 의미가 달라질 수 있다. DB가 과거 날짜 저장 자체를 막으면 관리자 보정, 이관, 과거 데이터 적재까지 제한할 수 있다. 이런 규칙은 도메인 또는 Service에서 판단한다.

## 5. Service 사전 조회가 DB 제약을 대체하지 못하는 이유

다음과 같은 Service 검사가 있다고 가정한다.

```text
중복 예약 존재 여부 조회
→ 없으면 저장
```

동시 요청에서는 두 요청이 모두 “없음”을 확인할 수 있다.

```text
요청 A: 중복 없음 확인
요청 B: 중복 없음 확인
요청 A: 저장
요청 B: 저장
```

Service 사전 조회만 있다면 중복 데이터가 만들어진다. DB 유일성 제약이 있으면 두 번째 저장 중 하나가 실패해 최종 무결성이 유지된다.

이 문제는 조회와 저장 사이의 경쟁 조건이다. 트랜잭션을 사용한다는 사실만으로 사라지지 않는다. 격리 수준, 잠금, 제약조건을 함께 봐야 한다.

**판단: 동시성에서 깨지면 안 되는 규칙은 DB 제약 또는 그에 준하는 잠금 전략으로 보장한다.**

## 6. Service 사전 조회를 추가하는 조건

DB 제약과 예외 변환만으로 충분한 경우에는 Service 사전 조회를 추가하지 않는다.

### 추가하는 경우

- 사용자가 선택할 수 있는 대안을 함께 안내해야 함
- 실패 전에 비용이 큰 외부 작업을 막아야 함
- 같은 DB 제약 위반이라도 클라이언트의 다음 행동이 달라짐
- 오류 메시지에 현재 가능한 상태를 포함해야 함
- 사전 조회 결과가 이후 유스케이스에도 실제로 필요함

예시:

```text
“이미 예약이 있습니다.”
```

보다 다음 안내가 제품 요구사항으로 필요할 때 Service 조회를 검토한다.

```text
“10시는 이미 예약됐습니다. 11시와 14시는 예약할 수 있습니다.”
```

### 추가하지 않는 경우

- DB 충돌을 `409 Conflict`와 안정적인 메시지로 변환하면 충분함
- 사전 조회가 저장 외에는 사용되지 않음
- 모든 제약조건을 기계적으로 Service에 복제하려는 목적
- 사전 조회가 무결성을 보장한다고 오해하는 경우

Service 사전 조회를 추가해도 저장 시점의 DB 예외 처리는 제거하지 않는다. 사전 조회 뒤 다른 요청이 데이터를 변경할 수 있기 때문이다.

## 7. DB 예외 변환

PR의 `JdbcReservationRepository`는 저장과 수정 중 `DuplicateKeyException`을 잡아 `DuplicatedException`으로 변환한다. HTTP 경계에서는 이를 `409 Conflict`로 반환한다.

```text
DB UNIQUE 위반
→ DuplicateKeyException
→ 애플리케이션 DuplicatedException
→ HTTP 409 Conflict
```

이 구조는 DB 제약을 최종 방어선으로 사용하면서 클라이언트에게 기술 예외를 노출하지 않는다.

### 변환 기준

- SQL과 제약조건 이름을 사용자에게 노출하지 않음
- 클라이언트 메시지와 서버 로그 정보를 분리
- 원본 예외를 cause로 보존
- 예상한 제약 위반만 해당 비즈니스 예외로 변환
- 같은 상위 예외에 여러 원인이 있으면 제약조건 이름·DB 오류 코드로 구분

`DataIntegrityViolationException`처럼 넓은 예외를 모두 “사용 중인 자원”으로 변환하면 `NOT NULL`, `CHECK`, 다른 FK 위반까지 잘못 해석할 수 있다. 원인을 구분할 수 없으면 무리하게 구체적인 비즈니스 예외로 변환하지 않는다.

## 8. 데이터 무결성과 사용자 안내의 우선순위

구현 순서는 다음과 같다.

1. DB 제약으로 최종 무결성을 보장한다.
2. DB 예외를 안정적인 애플리케이션 오류로 변환한다.
3. 클라이언트가 다음 행동을 결정할 수 있는 최소 메시지를 제공한다.
4. 제품 요구사항이 생기면 Service 사전 조회와 구체적인 대안을 추가한다.

사용자 친화적 안내가 무결성보다 먼저 구현돼서는 안 된다. 반대로 DB 오류 원문만 반환하는 것도 허용하지 않는다.

## 9. 중복 방지와 멱등성의 구분

유일성 제약은 동일한 상태의 중복 생성을 막는다. 멱등키는 같은 요청의 재전송을 식별해 같은 처리 결과를 재사용하는 장치다.

- 예약 슬롯 중복 방지: 복합 `UNIQUE`로 해결 가능
- 결제처럼 동일 요청의 부수 효과가 두 번 발생하면 안 됨: 멱등키 검토
- 같은 예약 요청에 기존 예약을 반환할지 `409`를 반환할지: API 의미와 제품 정책으로 결정

**판단: 유일성 제약으로 요구사항이 충족되면 멱등키를 추가하지 않는다.** 두 요청을 같은 요청으로 식별하고 이전 응답을 재사용해야 할 때만 도입한다.

## 10. 검증 방법

### Payload 테스트

- 필수 필드 누락
- 잘못된 JSON·날짜 형식
- 빈 문자열과 길이 경계
- 실패 시 `400`과 안정적인 오류 형식

### 도메인·Service 테스트

- 과거와 현재의 시간 경계
- 존재하지 않는 연관 자원
- 소유자와 비소유자
- 허용·거부되는 상태 전이
- Service 사전 조회가 있다면 조회 결과별 안내와 저장 호출 여부

### Repository 통합 테스트

- 동일 복합 키를 순차 저장할 때 두 번째 저장 실패
- FK·`NOT NULL`·`CHECK` 제약 위반
- 예상한 DB 예외가 애플리케이션 예외로 변환되는지 확인
- 제약 위반 이후 트랜잭션 상태 확인

### 동시성 통합 테스트

1. 서로 다른 트랜잭션에서 같은 예약 슬롯 저장을 동시에 시도한다.
2. 하나만 성공하는지 확인한다.
3. 최종 예약 개수가 1개인지 확인한다.
4. 실패 요청이 안정적인 충돌 예외로 변환되는지 확인한다.

PR에서는 복합 유일성 제약, 순차 중복 저장의 `DuplicatedException`, HTTP `409`를 확인했다. 실제 동시 요청 테스트는 확인하지 못했으므로 Race Condition 방어의 실행 검증은 **확인 필요**다.

## 최종 판단 기준

> Payload는 요청 형식을 검증하고, 도메인과 Service는 비즈니스 의미를 검증하며, DB는 동시 요청에서도 최종 무결성을 보장한다. Service 사전 조회는 구체적인 사용자 안내나 실패 전 비용 절감이 필요할 때만 추가한다. 사전 조회는 DB 제약을 대체하지 않으며, 모든 DB 제약을 Service에 중복 구현하지 않는다.

## 근거

- [방탈출 사용자 예약 PR #451](https://github.com/woowacourse/spring-roomescape-member/pull/451)
- [Service 검증과 DB 제약의 목적 차이 리뷰](https://github.com/woowacourse/spring-roomescape-member/pull/451#discussion_r3248710202)
- [요청 유효성과 데이터 필터링의 구분 리뷰](https://github.com/woowacourse/spring-roomescape-member/pull/451#discussion_r3248554155)
- [Repository와 Service 테스트 책임 리뷰](https://github.com/woowacourse/spring-roomescape-member/pull/451#discussion_r3248613027)
- [DB 무결성 예외 변환 제약 리뷰](https://github.com/woowacourse/spring-roomescape-member/pull/451#discussion_r3248476015)
- [최종 ReservationRequest](https://github.com/kangrae-jo/spring-roomescape-member/blob/1792fad08049df6acad5a1c57897bb1e065f80b6/src/main/java/roomescape/reservation/payload/ReservationRequest.java)
- [최종 ReservationService](https://github.com/kangrae-jo/spring-roomescape-member/blob/1792fad08049df6acad5a1c57897bb1e065f80b6/src/main/java/roomescape/reservation/service/ReservationService.java)
- [최종 JdbcReservationRepository](https://github.com/kangrae-jo/spring-roomescape-member/blob/1792fad08049df6acad5a1c57897bb1e065f80b6/src/main/java/roomescape/reservation/repository/JdbcReservationRepository.java)
- [최종 DB 스키마](https://github.com/kangrae-jo/spring-roomescape-member/blob/1792fad08049df6acad5a1c57897bb1e065f80b6/src/main/resources/schema.sql)

## 후속 검증

- [ ] 실제 동시 요청으로 복합 유일성 제약과 예외 변환을 검증한다.
- [ ] 운영 DB에서 제약 위반 예외와 제약조건 식별 방법을 확인한다.
- [ ] 구체적 사용자 안내가 필요한 사례에서 Service 사전 조회의 비용과 효과를 비교한다.
