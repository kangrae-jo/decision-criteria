# JPA 엔티티의 Aggregate 참조 방식

- 상태: `adopted`
- 주제: Aggregate 경계, ID 참조, JPA 엔티티 연관관계
- 선택: 다른 Aggregate는 ID로만 참조

## 결론

**채택: 다른 Aggregate는 도메인에서 ID VO, JPA Entity에서 원시 ID로 참조한다.** JPA Entity는 다른 Entity 타입을 필드로 보관하지 않는다.

**거부: `@ManyToOne`, `@OneToOne`, `@OneToMany`, `@ManyToMany`으로 Aggregate 사이를 연결한다.** 조회 편의를 위해 쓰기 모델의 탐색 범위와 결합을 넓히지 않는다.

**채택: ReservationTime은 Reservation과 별도 Aggregate다.** ReservationTime은 독립적으로 관리되고 여러 Reservation이 공유하므로 Reservation은 `reservationTimeId`만 가진다.

## 빠른 기준

| 질문 | 판단 |
| --- | --- |
| 대상이 독립적으로 생성·조회·삭제되는가? | 별도 Aggregate |
| 여러 Aggregate가 같은 대상을 공유하는가? | ID 참조 |
| 도메인에서 다른 Aggregate를 가리키는가? | 의미가 구분되는 ID VO |
| JPA Entity에서 다른 Aggregate를 가리키는가? | 원시 ID 컬럼 |
| 함께 조회해야 하는가? | 조회 쿼리 또는 조회 모델에서 조합 |
| 객체 탐색이 편리한가? | 연관관계 채택 근거로 사용하지 않음 |

## 1. Reservation과 ReservationTime

ReservationTime은 예약 없이도 생성·조회·비활성화할 수 있고 여러 Reservation이 공유한다. 두 객체의 생명주기가 다르므로 Aggregate 경계를 나눈다.

```text
Reservation Aggregate
└── reservationTimeId: ReservationTimeId

ReservationTime Aggregate
└── id: ReservationTimeId
```

Reservation이 시간대의 존재 여부와 활성 상태를 직접 검증하지 않는다. 예약 생성 유스케이스가 ReservationTime을 조회한 뒤 Reservation을 생성한다.

## 2. 도메인과 Entity의 표현

```java
public final class Reservation {

    private final ReservationTimeId reservationTimeId;
}

@Entity
public class ReservationEntity {

    @Column(name = "reservation_time_id", nullable = false)
    private Long reservationTimeId;
}
```

ReservationEntity에는 `ReservationTimeEntity` 필드와 연관관계 애너테이션을 두지 않는다. DB 외래 키는 사용할 수 있다. 외래 키 무결성과 객체 연관관계 매핑은 별개의 선택이다.

## 3. 조회 책임

예약과 시간 정보가 함께 필요한 조회는 Entity 연관관계 대신 명시적인 조회 계획을 사용한다.

- 화면 전용 데이터: DTO Projection 또는 Query Repository
- Aggregate 행위: 각각의 Repository로 조회한 뒤 Application Service에서 조합
- 대량 목록: SQL JOIN으로 필요한 필드만 조회

조회 편의를 이유로 Entity 연관관계를 추가하지 않는다. 구체적인 조회 방식은 [JPA 조회 계획과 N+1](jpa-fetch-plan.md)에서 다룬다.

## 4. 시간 변경과 삭제

**거부: 기존 ReservationTime의 시간 값을 수정한다.** 시간 변경은 새로운 ReservationTime을 생성하는 방식으로 표현한다.

- 참조하는 Reservation이 없으면 삭제할 수 있다.
- 기존 Reservation이 참조하면 물리 삭제하지 않고 비활성화한다.
- 비활성화한 시간은 신규 예약 대상에서 제외하되 기존 예약 조회에는 사용한다.

## 5. 트레이드오프

| 효과 | 비용·제약 |
| --- | --- |
| Aggregate 사이의 탐색과 변경 범위가 명시적임 | `reservation.getTime()` 탐색 불가 |
| JPA 프록시와 연관관계 동기화에 의존하지 않음 | 필요한 Aggregate를 직접 조회해야 함 |
| 목록 조회 SQL을 유스케이스별로 설계할 수 있음 | 조회용 쿼리와 DTO가 늘어날 수 있음 |
| Entity가 단순 ID 값을 보관함 | 변환 코드와 외래 키 관리 필요 |

## 검증 방법

- Entity의 다른 Entity 타입 필드와 연관관계 애너테이션이 0건인지 검사한다.
- 도메인의 다른 Aggregate 참조가 ID VO인지 검사한다.
- DB 외래 키가 ID 컬럼의 무결성을 보장하는지 확인한다.
- 비활성화된 ReservationTime으로 신규 예약할 수 없는지 테스트한다.
- 기존 Reservation이 비활성화된 ReservationTime을 계속 조회할 수 있는지 테스트한다.

## 최종 판단 기준

> 생명주기가 다르거나 여러 Aggregate가 공유하는 대상은 ID로 참조한다. 도메인은 ID VO를, JPA Entity는 원시 ID 컬럼을 사용한다. 필요한 데이터 조합은 조회 쿼리나 Application Service가 담당하며, 조회 편의를 위해 Entity 연관관계를 추가하지 않는다.
