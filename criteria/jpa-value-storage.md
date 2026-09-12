# JPA 값 저장 방식

- 상태: adopted
- 예시 기준: DeliveryEntity의 값 평탄화
- 주제: 원시 컬럼, 변환 비용, Embeddable, Enumerated
- 선택: DeliveryEntity는 단순 값만 저장

## 결론

**채택: DeliveryEntity는 Long, String, Integer, Instant 같은 단순 값만 보관한다.** 다른 Entity 참조와 Embedded 매핑을 만들지 않는다.

**채택: 도메인 VO는 from()에서 원시 값으로 풀고, toDomain()에서 다시 조합한다.** Order는 Long orderId로만 저장한다.

**거부: DeliveryRoute, FurnitureInfo, DeliveryAssignment를 Embeddable로 저장한다.** 현재 선택의 비용은 변환 코드와 왕복 변환 테스트다.

## 빠른 기준

| 질문 | 판단 |
| --- | --- |
| 다른 Aggregate를 참조하는가? | Long ID 컬럼만 저장 |
| 도메인 값이 여러 값을 묶는가? | 현재 DeliveryEntity에서는 단순 컬럼으로 평탄화 |
| 두 컬럼이 함께 있어야 유효한가? | 모두 존재하거나 모두 null인지 검증 |
| enum 이름이 DB 저장 계약인가? | Enumerated(EnumType.STRING) 허용 |
| enum 이름이 구현 세부 사항인가? | String과 명시적 변환 사용 |
| Embeddable이 변환 비용을 줄이는가? | 별도 모델에서만 검토 |

## 1. 적용 범위

이 기준은 JPA 엔티티가 도메인 값을 어떤 DB 컬럼 표현으로 저장할지 결정한다. Aggregate의 상태 전이, VO의 불변식, 조회 계획은 비범위다.

~~~text
Domain Delivery
├── orderId: OrderId
├── route: DeliveryRoute
└── assignment: DeliveryAssignment?

Persistence DeliveryEntity
├── orderId: Long
├── pickupAddress: String
├── deliveryAddress: String
├── assignedDriverId: Long?
└── acceptedAt: Instant?
~~~

## 2. DeliveryEntity 평탄화

DeliveryEntity의 필드는 단순 값만 가진다. DeliveryRoute, FurnitureInfo, EstimatedDeliveryFee, DeliveryAssignment를 필드 타입으로 보관하지 않는다.

~~~java
@Entity
@Table(name = "delivery")
@NoArgsConstructor(access = AccessLevel.PROTECTED)
public class DeliveryEntity {

    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    @Column(name = "order_id", nullable = false, unique = true)
    private Long orderId;
    @Column(name = "item_name", nullable = false) private String itemName;
    @Column(nullable = false) private String category;
    @Column(name = "pickup_address", nullable = false) private String pickupAddress;
    @Column(name = "delivery_address", nullable = false) private String deliveryAddress;
    @Column(name = "pickup_phone_number", nullable = false) private String pickupPhoneNumber;
    @Column(name = "delivery_phone_number", nullable = false) private String deliveryPhoneNumber;
    @Column(name = "estimated_fee", nullable = false)
    private Integer estimatedDeliveryFee;
    @Column(name = "assigned_driver_id") private Long assignedDriverId;
    @Column(name = "accepted_at") private Instant acceptedAt;
    @Column(nullable = false, length = 20)
    private String status;
    @Column(name = "requested_at", nullable = false, updatable = false)
    private Instant requestedAt;
    @Column(name = "picked_up_at") private Instant pickedUpAt;
    @Column(name = "delivered_at") private Instant deliveredAt;

    // 기본 생성자와 전체 생성자 생략
}
~~~

| 도메인 값 | 저장 컬럼 |
| --- | --- |
| OrderId | Long orderId |
| FurnitureInfo | String itemName, String category |
| DeliveryRoute | 주소·연락처 String 4개 |
| EstimatedDeliveryFee | Integer estimatedDeliveryFee |
| DeliveryAssignment | Long assignedDriverId, Instant acceptedAt |
| DeliveryStatus | String status |

## 3. 변환 책임과 비용

변환은 영속성 계층이 소유한다. 도메인은 DB 컬럼명, 기본 생성자, null 복원 규칙을 모른다.

~~~java
public static DeliveryEntity from(Delivery delivery) {
    DeliveryAssignment assignment = delivery.assignment();
    return new DeliveryEntity(
            delivery.id() == null ? null : delivery.id().value(), delivery.orderId().value(),
            delivery.furnitureInfo().itemName(), delivery.furnitureInfo().category(),
            delivery.route().pickupAddress().value(), delivery.route().deliveryAddress().value(),
            delivery.route().pickupPhoneNumber().value(), delivery.route().deliveryPhoneNumber().value(),
            delivery.estimatedDeliveryFee().value(),
            assignment == null ? null : assignment.driverId().value(),
            assignment == null ? null : assignment.acceptedAt(),
            delivery.status().name(), delivery.requestedAt(), delivery.pickedUpAt(), delivery.deliveredAt());
}

public Delivery toDomain() {
    return Delivery.restore(new DeliveryState(
            id == null ? null : new DeliveryId(id), new OrderId(orderId),
            new FurnitureInfo(itemName, category),
            new DeliveryRoute(
                    new Address(pickupAddress), new Address(deliveryAddress),
                    new PhoneNumber(pickupPhoneNumber), new PhoneNumber(deliveryPhoneNumber)),
            new EstimatedDeliveryFee(estimatedDeliveryFee),
            toDomainAssignment(), toDomainStatus(status),
            requestedAt, pickedUpAt, deliveredAt));
}

private DeliveryAssignment toDomainAssignment() {
    if (assignedDriverId == null && acceptedAt == null) {
        return null;
    }
    return new DeliveryAssignment(new DriverId(assignedDriverId), acceptedAt);
}

private static DeliveryStatus toDomainStatus(String storedStatus) {
    try {
        return DeliveryStatus.valueOf(storedStatus);
    } catch (IllegalArgumentException exception) {
        throw new IllegalStateException("저장된 배송 상태가 유효하지 않습니다: " + storedStatus, exception);
    }
}
~~~

| 효과 | 비용 |
| --- | --- |
| Entity의 저장 구조가 모든 컬럼으로 명확함 | VO마다 값 복사 코드 필요 |
| 다른 Aggregate 접근이 ID 값으로 한정됨 | 필드 추가·이름 변경 시 양방향 변환 수정 필요 |
| JPA 복합 값 매핑에 의존하지 않음 | 매핑 누락을 왕복 변환 테스트로 찾아야 함 |

assignedDriverId와 acceptedAt은 둘 다 null이면 미배정이다. 하나만 null이면 DeliveryAssignment로 복원할 수 없으므로 오류다.

## 4. Embeddable 판단

Embeddable은 독립 영속 식별자 없이 소유 엔티티의 일부로 저장하는 복합 값 매핑이다. 도메인 VO와 자주 대응하지만 같은 개념은 아니다.

| 상황 | 판단 |
| --- | --- |
| 현재 DeliveryEntity의 DeliveryRoute, FurnitureInfo, DeliveryAssignment | 거부. 단순 컬럼으로 평탄화 |
| 다른 모델에서 여러 컬럼이 함께 움직이고 평탄화 비용이 과도함 | 영속성 전용 Embeddable 검토 |
| 독립 ID·Repository·상태 전이가 필요함 | Embeddable 거부. 엔티티 검토 |
| 순수 도메인 타입에 Embeddable을 붙여야 함 | 거부 |

**판단: 도메인 VO의 불변식이 Embeddable 사용을 강제하지 않는다.** 현재 선택에서는 from()과 toDomain()이 값 조합을 복원한다.

## 5. Enumerated 판단

Jakarta Persistence의 enum 기본 저장 방식은 ORDINAL이다. STRING은 enum 상수 이름을 저장한다.

| 상황 | 판단 |
| --- | --- |
| enum 상수명이 DB의 공식 저장값 | Enumerated(EnumType.STRING) 허용 |
| DB 코드가 enum 이름과 다름 | String과 명시적 코드 변환 |
| enum 이름 변경·값 제거 | DB 마이그레이션과 이전 값 호환 변환 |
| Enumerated 옵션 생략 | 거부. 기본 ORDINAL에 의존하지 않음 |
| Enumerated(EnumType.ORDINAL) | 거부. 상수 순서 변경이 저장 의미를 바꿀 수 있음 |

현재 DeliveryEntity는 String status와 name()·valueOf() 변환을 사용한다. 상태 이름이 DB 계약으로 확정된 경우에만 Enumerated(EnumType.STRING)을 선택한다.

## 6. 검증 방법

- Delivery → DeliveryEntity → Delivery 왕복 뒤 ID, VO 구성 값, 상태, 시각이 보존되는지 확인한다.
- order_id가 다른 Aggregate의 ID 값으로 저장되는지 확인한다.
- assignedDriverId·acceptedAt의 전체 null, 정상 값, 부분 null을 검증한다.
- 알 수 없는 상태 문자열과 enum 이름 변경 시의 데이터 호환을 검증한다.
- 필드 추가·이름 변경 때 from(), toDomain(), 왕복 변환 테스트가 함께 수정되는지 확인한다.

## 최종 판단 기준

> DeliveryEntity는 단순 DB 값만 보관한다. 도메인 VO는 영속성 계층의 from()과 toDomain()에서 평탄화·복원한다. 다른 Aggregate는 ID 값으로만 저장한다. Embeddable은 일반 선택지이지만 현재 DeliveryEntity에는 사용하지 않으며, enum 저장값은 기본 ORDINAL에 맡기지 않는다.

## 검증 범위

- 확인 필요: 실제 스키마 적용 뒤 변환 코드량과 변경 비용
- 비범위: Aggregate 상태 전이와 VO 자체의 불변식 설계
