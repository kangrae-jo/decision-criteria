# SQL과 Service 로직의 구분

- 상태: `validated`
- 출처: [방탈출 사용자 예약 PR #373](https://github.com/woowacourse/spring-roomescape-member/pull/373)의 구현과 코드 리뷰
- 검토 기준: PR 최종 커밋 `0070ac6`
- 주제: DB 연산, 도메인 해석, Service 책임, Repository 소유권

## 결론

**채택: 데이터를 줄이는 관계 연산은 DB가 담당한다.** 필터링, 집계, 정렬, 제한, 존재 여부, 집합 연산을 위해 전체 데이터를 애플리케이션으로 가져오지 않는다.

**채택: 데이터의 의미를 해석하는 판단은 도메인 또는 Service가 담당한다.** 사용자 등급, 도메인 상태, 외부 API 결과, 시간 기준처럼 DB가 스스로 알 수 없는 맥락을 SQL에 숨기지 않는다.

**채택: Service가 맥락을 쿼리 파라미터로 결정하고, Repository가 관계 연산을 수행하는 조합을 우선한다.** SQL과 Service 중 한쪽에 모든 책임을 몰아넣는 문제가 아니다.

## 빠른 기준

| 작업 | 우선 위치 | 근거 |
| --- | --- | --- |
| 조건에 맞는 행 필터링 | DB | 필요한 데이터만 전송 |
| 집계·그룹화 | DB | 저장된 집합에 대한 관계 연산 |
| 정렬·상위 N개·페이지 제한 | DB | 전체 데이터를 메모리에 올리지 않음 |
| 존재 여부·중복 여부 | DB 조회 또는 DB 제약 | 인덱스와 제약조건 활용 |
| 교집합·차집합·합집합 | DB | 집합 연산을 데이터가 있는 위치에서 처리 |
| 사용자 등급에 따른 정책 | 도메인 또는 Service | 사용자 맥락 해석 필요 |
| 도메인 상태에 따른 행위 가능 여부 | 도메인 | 불변식과 상태 전이 보호 |
| 여러 도메인·외부 API 결과 조합 | Service | 유스케이스 조율 필요 |
| “오늘”과 조회 기간 결정 | Service | 시간대와 비즈니스 기준 해석 |
| 결정된 기간의 필터·집계 | DB | 관계 연산에 해당 |

## 1. 복잡한 SQL과 복잡한 Service의 양자택일이 아니다

SQL과 Service의 코드 줄 수를 비교해서 위치를 결정하지 않는다. 먼저 작업의 성격을 분리한다.

1. Service 또는 도메인이 비즈니스 맥락을 해석한다.
2. 해석 결과를 날짜, 상태, 식별자, 제한 개수 같은 명시적 파라미터로 만든다.
3. Repository가 파라미터를 사용해 필터링·집계·정렬한다.
4. 조회 결과에 추가 정책 판단이 필요하면 도메인 또는 Service가 해석한다.

예를 들어 “최근 7일 인기 테마 10개”는 하나의 책임이 아니다.

- Service: `Clock`을 사용해 오늘과 조회 시작일을 결정
- Repository: 기간 필터링, 예약 수 집계, 내림차순 정렬, 10개 제한

이렇게 나누면 비즈니스 시간 기준은 테스트할 수 있고, 대량 데이터 축소는 DB가 수행한다.

## 2. DB가 담당하는 일: 데이터를 줄이는 연산

다음 연산은 DB에 우선 배치한다.

- `WHERE` 조건 필터링
- `JOIN`을 통한 관계 결합
- `COUNT`, `SUM` 같은 집계
- `GROUP BY`
- `ORDER BY`
- `LIMIT`, 페이지네이션
- `EXISTS`
- 교집합·차집합·합집합

### 예약 가능 시간 사례

초기 구현은 다음 순서였다.

```text
전체 예약 시간 조회
→ 예약된 시간 ID 조회
→ Java에서 List.contains()로 제외
```

최종 구현은 `ReservationTimeRepository.findAvailableTimesByDateAndThemeId()`가 DB에서 전체 시간과 예약된 시간의 차집합을 구한다.

```text
reservation_time
− 해당 날짜·테마에 예약된 time_id
= 예약 가능한 시간
```

**판단: 이 연산은 도메인 상태 해석이 아니라 저장된 집합의 차집합이므로 DB에 둔다.**

장점:

- 필요한 행만 애플리케이션으로 전송
- 애플리케이션 메모리 사용 감소
- 인덱스와 DB 실행 계획 활용 가능
- 여러 쿼리 결과를 Java에서 조립하는 비용 제거
- 여러 개별 조회 사이에서 데이터가 바뀔 수 있는 구간 축소

### 제약

- 데이터가 적으면 성능 차이가 보이지 않을 수 있다.
- SQL이 복잡해질수록 DB 종속성과 유지보수 비용이 증가한다.
- `NOT IN`, `NOT EXISTS`, anti join은 NULL 가능성과 실행 계획을 확인해 선택해야 한다.
- “DB가 더 빠르다”는 일반론만으로 판단하지 않는다. 데이터 규모와 실행 계획으로 검증한다.

## 3. 도메인과 Service가 담당하는 일: 맥락 해석

다음 판단은 SQL보다 도메인 또는 Service에 우선 배치한다.

- VIP 회원에게만 예외적으로 허용되는 행위
- 예약 상태에 따라 가능한 다음 행동
- 사용자 권한과 소유권 판단
- 외부 API 결과와 내부 상태의 조합
- 시간대에 따른 “오늘”의 결정
- 여러 Aggregate의 행위를 조율하는 유스케이스

위 규칙은 데이터 필터 조건처럼 보일 수 있지만, 변경 이유가 비즈니스 정책에 있다. SQL에 직접 넣으면 정책을 찾기 어렵고 DB 조회와 도메인 규칙이 함께 변경된다.

대량 데이터에서 정책 대상까지 DB로 줄여야 한다면 Service가 적용할 정책과 조건을 명시적 파라미터로 결정하고 Repository가 필터링한다. DB가 조건을 실행하더라도 정책의 의미와 변경 책임까지 Repository로 넘기지는 않는다.

### 도메인과 Service의 구분

- 한 객체의 상태와 값만으로 판단 가능: 도메인 객체
- 여러 객체·Repository·외부 시스템을 조율: Service
- 대량 후보를 줄이는 일: Repository
- 후보를 가져온 뒤 개별 객체의 행위 가능 여부 판단: 도메인

### 인기 테마 사례

PR 최종 구현은 다음처럼 분리돼 있다.

```text
ThemeService
→ Clock으로 today 결정
→ startDate와 endDate 계산
→ ThemeRepository에 기간과 limit 전달

ThemeRepository
→ 기간 필터링
→ 테마별 예약 수 집계
→ 인기순 정렬
→ limit 적용
```

“한국 표준시 기준 오늘” 같은 시간 기준은 비즈니스 맥락이다. Service가 하나의 시간 기준을 결정하고 DB에는 값으로 전달한다. SQL의 `CURRENT_DATE`와 애플리케이션 시간이 섞이는 문제도 피할 수 있다.

## 4. Repository 위치 결정

최종 반환 타입만으로 Repository 위치를 결정하지 않는다. 다음 질문을 순서대로 확인한다.

1. 호출자가 요청하는 핵심 자원은 무엇인가?
2. 어떤 Aggregate 또는 읽기 모델을 반환하는가?
3. 쿼리가 읽는 테이블은 핵심 자원인가, 필터 조건을 제공하는 보조 자원인가?
4. 이 메서드를 어느 Repository 포트에 두면 Service의 패키지 의존성이 단순해지는가?
5. 쿼리가 여러 도메인의 정책을 하나의 Repository에 숨기고 있지는 않은가?

### 예약 가능 시간

- 요청 자원: 예약 가능한 `ReservationTime`
- 반환 타입: `List<ReservationTime>`
- `reservation` 테이블의 역할: 이미 점유된 시간 식별
- 최종 위치: `ReservationTimeRepository`

초기 `ReservationTimeService`는 `ReservationTimeRepository`와 `ReservationRepository`에 모두 의존해 두 결과를 조립했다. 최종 구현에서는 가용 시간 조회 포트를 `ReservationTimeRepository`에 두어 Service의 교차 패키지 의존성을 줄였다.

현재 규모에서는 **채택**한다.

### 인기 테마

- 요청 자원: 인기순 `Theme`
- 반환 타입: `List<Theme>`
- `reservation` 테이블의 역할: 인기도 집계 근거
- 최종 위치: `ThemeRepository`

예약 테이블을 읽는다는 이유만으로 `ReservationRepository`에 둘 필요는 없다. 쿼리의 목적은 예약 반환이 아니라 테마 읽기 모델 생성이다.

### 별도 Query Repository가 필요한 경우

다음 조건이 커지면 기존 Aggregate Repository에 억지로 넣지 않는다.

- 여러 도메인의 필드를 조합한 전용 Projection 반환
- 화면별 읽기 모델이 다수 발생
- 한 Repository가 다른 도메인의 조회 정책을 계속 흡수
- 쿼리 변경 이유가 Aggregate 저장 책임과 무관
- 패키지 순환 의존 또는 다수 Service 의존 발생

이 경우 `ReservationTimeQueryRepository`, `AvailabilityQueryPort` 같은 읽기 전용 포트를 별도로 둔다.

**판단: Repository 소유권은 반환 타입, 조회 목적, 사용 테이블, 패키지 결합도를 함께 보고 결정한다.**

## 5. 복잡한 SQL을 허용하는 기준

SQL이 길다는 이유만으로 Service로 옮기지 않는다. 다음 조건을 만족하면 복잡한 SQL도 허용한다.

- 연산의 핵심이 관계 대수로 설명됨
- 반환 타입과 Repository 계약이 명확함
- 쿼리 조건의 의미가 메서드 이름과 파라미터에 드러남
- Repository 통합 테스트로 경계값을 검증함
- 데이터 규모가 크다면 실행 계획과 인덱스를 확인함
- DB 종속 문법을 사용한 이유와 교체 비용을 감수할 수 있음

다음 상황이면 SQL에서 분리한다.

- 사용자 등급이나 도메인 상태에 따라 의미가 달라짐
- 외부 API 결과가 필요함
- 분기 규칙이 도메인 용어로 자주 변경됨
- SQL 하나가 여러 유스케이스의 정책을 동시에 책임짐
- 쿼리를 이해하려면 애플리케이션의 전체 실행 흐름을 알아야 함

## 6. Repository가 반환할 사실과 Service가 해석할 의미

Repository는 DB 작업 결과를 사실로 반환하고, Service는 그 사실을 비즈니스 의미로 해석한다.

PR 최종 구현의 삭제 흐름:

```text
Repository.deleteById(id)
→ 영향받은 행 수 반환

Service
→ 0이면 존재하지 않는 자원으로 해석
→ 비즈니스 예외 발생
```

**채택: “영향받은 행이 0개”는 DB 사실이고, “삭제할 자원이 없다”는 애플리케이션 의미다.**

단, Unique·Foreign Key 위반처럼 DB 제약에서 발생한 예외를 Repository Adapter가 기술 독립적인 예외로 변환하는 것은 허용한다. 변환된 예외가 Repository 포트의 계약에 드러나야 한다.

## 7. 검증 방법

### Repository 통합 테스트

- 예약된 시간만 정확히 제외되는지 확인
- 예약이 없을 때 전체 시간이 반환되는지 확인
- 기간 시작 포함·종료 제외 경계 확인
- 집계 결과의 정렬 순서 확인
- `limit` 적용 확인
- 결과가 없을 때 빈 목록 확인
- NULL과 중복 데이터 조건 확인

PR에서는 예약 가능 시간 차집합과 인기 테마 집계·정렬을 실제 Repository 테스트로 검증한다.

### Service 테스트

- `Clock`으로 오늘을 고정
- 조회 시작일과 종료일 계산 확인
- 사용자 등급과 상태에 따른 분기 확인
- Repository에 전달하는 파라미터 확인
- 외부 API 성공·실패 결과 조합 확인

### 성능 검증

- 예상 운영 데이터 규모의 Fixture 준비
- `EXPLAIN` 또는 `EXPLAIN ANALYZE`로 실행 계획 확인
- 전체 행 조회 수와 실제 반환 행 수 비교
- 인덱스 적용 여부 확인
- 쿼리 횟수와 애플리케이션 메모리 사용 비교

PR에서는 책임 분리와 결과 정확성을 확인했다. 대량 데이터 실행 계획과 성능 개선 수치는 확인하지 않았으므로 **확인 필요**다.

## 최종 판단 기준

> 필터링, 집계, 정렬, 제한, 차집합처럼 데이터를 줄이는 연산은 DB에서 수행한다. 사용자 등급, 도메인 상태, 외부 API 결과, 시간 기준처럼 맥락을 해석하는 판단은 도메인 또는 Service가 담당한다. Service가 맥락을 명시적 쿼리 파라미터로 만들고 Repository가 관계 연산을 수행하는 조합을 우선한다. Repository 위치는 최종 반환 타입뿐 아니라 조회 목적, 사용 자원, 패키지 결합도를 함께 보고 결정한다.

## 근거

- [방탈출 사용자 예약 PR #373](https://github.com/woowacourse/spring-roomescape-member/pull/373)
- [SQL과 Service 판단 기준 리뷰](https://github.com/woowacourse/spring-roomescape-member/pull/373#discussion_r3208111597)
- [Repository 위치와 패키지 결합도 리뷰](https://github.com/woowacourse/spring-roomescape-member/pull/373#discussion_r3208081455)
- [애플리케이션 시간과 DB 시간 기준 리뷰](https://github.com/woowacourse/spring-roomescape-member/pull/373#discussion_r3219476124)
- [최종 ReservationTimeService](https://github.com/kangrae-jo/spring-roomescape-member/blob/0070ac6062144f92fe870afc0742ca78d481d438/src/main/java/roomescape/reservationtime/service/ReservationTimeService.java)
- [최종 JdbcReservationTimeRepository](https://github.com/kangrae-jo/spring-roomescape-member/blob/0070ac6062144f92fe870afc0742ca78d481d438/src/main/java/roomescape/reservationtime/repository/JdbcReservationTimeRepository.java)
- [최종 ThemeService](https://github.com/kangrae-jo/spring-roomescape-member/blob/0070ac6062144f92fe870afc0742ca78d481d438/src/main/java/roomescape/theme/service/ThemeService.java)
- [최종 JdbcThemeRepository](https://github.com/kangrae-jo/spring-roomescape-member/blob/0070ac6062144f92fe870afc0742ca78d481d438/src/main/java/roomescape/theme/repository/JdbcThemeRepository.java)

## 후속 검증

- [ ] 운영 데이터 규모에서 가용 시간 쿼리 실행 계획을 확인한다.
- [ ] `NOT IN`과 `NOT EXISTS`의 실행 계획과 NULL 동작을 비교한다.
- [ ] 조회 요구가 커질 때 Aggregate Repository와 전용 Query Repository의 변경 비용을 비교한다.
