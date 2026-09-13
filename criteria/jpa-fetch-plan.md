# JPA 조회 계획과 N+1

- 상태: `revised`
- 예시: Reservation 목록 조회의 N+1과 EntityGraph 적용 기록
- 주제: FetchType, N+1, JOIN, 조회별 SQL

## 결론

**채택: FetchType을 조회 계획으로 간주하지 않는다.** 유스케이스별로 필요한 데이터와 실제 SQL을 확인하고 조회 방식을 명시한다.

**채택: ID 참조 Entity에서 연관 데이터가 필요한 목록은 명시적 JOIN과 DTO Projection 또는 Query Repository로 조회한다.** 조회 편의를 위해 Entity 연관관계를 추가하지 않는다.

**조건부 채택: 기존 연관관계 매핑을 사용하는 모델은 EntityGraph나 fetch join으로 조회별 로딩 범위를 명시한다.** 이는 현재의 ID 참조 선택을 변경하지 않는다.

## 빠른 기준

| 상황 | 선택 |
| --- | --- |
| ID 참조 Entity의 연관 데이터 목록 | 명시적 JOIN과 DTO Projection |
| 여러 Aggregate의 조회 결과 조합 | Query Repository 또는 Application Service |
| 기존 ToOne 연관관계를 함께 로딩 | EntityGraph 또는 fetch join |
| 여러 ToMany 컬렉션을 함께 로딩 | 행 증가와 페이징부터 확인 |
| 다수 프록시 초기화 | Batch Fetch 검토 |
| FetchType만 확인한 상태 | 성능 판단 보류 |

## 1. 적용 범위

이 문서는 Entity 참조 방식을 결정하지 않는다. Aggregate 경계와 ID 참조 선택은 [JPA 엔티티의 Aggregate 참조 방식](jpa-aggregate-reference.md)에서 다룬다.

포함:

- FetchType의 의미
- 목록 조회의 N+1
- JOIN에 따른 결과 행 증가
- 조회별 EntityGraph, fetch join, Projection 선택

비범위:

- ID 참조와 Entity 연관관계 중 무엇을 선택할지
- Aggregate 경계
- 도메인 모델과 JPA Entity의 분리

## 2. FetchType의 의미

`EAGER`는 연관 Entity가 함께 사용 가능한 상태로 로딩된다는 것만 보장한다.

- 한 번의 JOIN으로 가져오는 것을 보장하지 않는다.
- 추가 SELECT로 연관 Entity를 가져올 수 있다.
- 단건과 목록 조회에서 생성되는 SQL이 다를 수 있다.
- `@ManyToOne`과 `@OneToOne`의 JPA 기본 FetchType은 `EAGER`다.
- 기존 연관관계 매핑에 지연 로딩이 필요하면 `LAZY`를 명시한다.

**판단: FetchType만 보고 쿼리 수와 성능을 확정하지 않는다.**

## 3. N+1 예시

기존 기록에는 Reservation 3개가 각각 Time과 Theme을 참조하는 목록 조회에서 다음 SQL이 관찰됐다고 남아 있다.

```text
Reservation 목록 조회 1회
Time 조회 3회
Theme 조회 3회
총 SELECT 7회
```

이 기록에는 SQL 로그와 재현 코드가 포함돼 있지 않아 정확한 쿼리 수는 **확인 필요**다.

예시가 보여주는 문제는 목록 개수 N에 따라 연관 데이터 조회가 추가될 수 있다는 점이다. 정확한 쿼리 수는 영속성 컨텍스트, 배치 설정, 조회 방식에 따라 달라진다.

## 4. JOIN과 결과 행

여러 ToOne 관계를 조인하면 각 루트가 하나의 대상만 가리키므로 일반적으로 루트 행이 곱집합으로 증가하지 않는다.

여러 ToMany 컬렉션을 동시에 조인하면 결과 행이 증가할 수 있다.

```text
Reservation 1개
× Waitings 3개
× Payments 2개
= 결과 행 6개
```

ToMany JOIN을 목록 조회에 사용할 때는 중복 행, 메모리 사용, 페이징 정확성을 함께 확인한다.

## 5. ID 참조 Entity의 조회 계획

현재 선택에서는 ReservationEntity가 `reservationTimeId: Long`만 가진다. Reservation과 시간 정보가 함께 필요한 화면은 조회 쿼리에서 조합한다.

```sql
SELECT r.id,
       r.date,
       rt.start_at
FROM reservation r
JOIN reservation_time rt
  ON rt.id = r.reservation_time_id
WHERE r.reserver_id = :reserverId
```

- Entity 반환이 필요 없으면 DTO Projection을 사용한다.
- 복수 Aggregate를 위한 조회라면 전용 Query Repository를 둔다.
- Aggregate 행위가 필요하면 각 Repository로 조회하고 Application Service에서 조합한다.
- 조회 요구만으로 `@ManyToOne`을 추가하지 않는다.

## 6. 연관관계 매핑 예시의 조회 계획

기존 연관관계 매핑을 다루는 경우 EntityGraph나 fetch join을 사용할 수 있다.

```text
적용 전: Reservation 1회 + Time N회 + Theme N회
적용 후: Reservation LEFT JOIN Time LEFT JOIN Theme
```

EntityGraph는 Entity의 기본 FetchType을 변경하지 않고 특정 조회에서 포함할 연관관계를 지정한다. 연관관계가 없는 ID 참조 Entity에는 적용할 대상이 없으므로 명시적 JOIN이나 Projection을 사용한다.

## 7. 변경 후 재검증

조회 대상 필드나 참조 ID가 추가되면 기존 목록 조회 SQL을 다시 확인한다.

- SELECT 횟수
- JOIN 수와 결과 행 수
- Projection에 포함된 컬럼
- 정렬과 페이징 결과
- 인덱스 사용 여부

## 검증 방법

- 대표 단건·목록 조회의 SQL과 SELECT 횟수를 기록한다.
- 데이터 N건에서 쿼리 수가 N에 비례하는지 확인한다.
- ToMany JOIN 전후의 결과 행 수와 페이징 결과를 비교한다.
- 운영 데이터 규모에 가까운 조건에서 실행 계획과 인덱스를 확인한다.

## 예시의 후속 확인

- [ ] `findAllByReserver`의 SQL을 재현한다.
- [ ] 단건 조회와 목록 조회의 SQL 차이를 기록한다.
- [ ] 조회 필드 추가 전후의 SELECT 횟수를 비교한다.

## 최종 판단 기준

> FetchType은 기본 로딩 시점을 정할 뿐 조회 성능을 완성하지 않는다. ID 참조 Entity는 명시적 JOIN과 Projection으로 조회 데이터를 조합한다. 기존 연관관계 매핑을 다룰 때만 EntityGraph나 fetch join을 사용하며, 모든 조회는 실제 SQL과 결과 행을 기준으로 확인한다.
