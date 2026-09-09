# 도메인 모델과 JPA 엔티티 분리

- 상태: draft
- 예시 기준: 배송 Aggregate Delivery, VO DeliveryId·OrderId·DriverId·DeliveryRoute·FurnitureInfo·EstimatedDeliveryFee
- 주제: 순수 도메인 모델, JPA 엔티티, 변환 책임, Enumerated, Embeddable
- 선택: 도메인 모델과 JPA 엔티티 분리

## 결론

**채택: 비즈니스 규칙을 가진 도메인 모델과 DB 매핑을 위한 JPA 엔티티를 별도 클래스로 둔다.** 도메인 모델은 JPA·Hibernate·Spring Data 애너테이션과 타입을 import하지 않는다.

**채택: 영속성 계층이 도메인 모델을 변환한다.** DeliveryEntity.from(delivery)과 toDomain()처럼 엔티티 또는 영속성 매퍼가 변환을 담당한다. 데이터는 양방향으로 이동해도 코드 의존성은 Persistence → Domain 단방향으로 유지한다.

**채택: 다른 Aggregate는 두 계층 모두 ID만 보관한다.** 도메인은 OrderId처럼 의미가 구분되는 VO를, JPA 엔티티는 Long orderId 같은 컬럼을 사용한다.

**거부: 도메인 모델에 Entity, Embeddable, Enumerated 같은 JPA 매핑 정보를 둔다.** JPA를 포기하는 선택이 아니라, 기술 요구사항이 비즈니스 모델을 바꾸지 못하게 하는 선택이다.

## 빠른 기준

| 질문 | 판단 |
| --- | --- |
| JPA 없이도 생성·변경·검증돼야 하는 비즈니스 규칙인가? | 순수 도메인 모델 |
| 테이블·컬럼·기본 생성자·영속화 표현을 다루는가? | JPA 엔티티 또는 영속성 매퍼 |
| 다른 Aggregate를 참조하는가? | 도메인과 JPA 엔티티 모두 ID로 참조 |
| 조회 모델을 설계하는가? | 비범위. 이 기준에 포함하지 않음 |
| enum 이름이 DB의 공식 저장값으로 확정됐는가? | 엔티티에서만 Enumerated(EnumType.STRING) 허용 가능 |
| enum 저장값이 별도 코드이거나 변경될 수 있는가? | 엔티티의 String과 명시적 변환 함수 |
| 값 묶음에 독립 ID·Repository·상태 전이가 필요한가? | Embeddable 거부. 엔티티 검토 |
| 값 묶음이 소유 엔티티에 종속되고 여러 컬럼을 함께 가지는가? | 영속성 전용 Embeddable 검토 |

## 1. 적용 범위

이 기준은 도메인 모델과 JPA 엔티티의 분리, 변환 책임, ID 저장 방식을 다룬다. 조회 계획은 비범위다.

**채택: 다른 Aggregate는 두 계층 모두 ID만 보관한다.** 도메인은 의미가 구분되는 ID VO를, JPA 엔티티는 해당 ID의 원시 컬럼을 사용한다.

~~~text
Domain Delivery
└── orderId: OrderId

Persistence DeliveryEntity
└── orderId: Long
~~~

조회에 Order 정보가 필요해도 DeliveryEntity에 Order Entity를 보관하지 않는다. 조회 모델·Projection·명시적 조회는 별도 판단 기준에서 다룬다.

## 2. 두 방식의 트레이드오프

| 항목 | 통합 모델: 도메인 = JPA 엔티티 | 분리 모델: 도메인 ≠ JPA 엔티티 — 현재 선택 |
| --- | --- | --- |
| 클래스 수 | 적음 | 도메인·엔티티·변환 코드 증가 |
| 변경 | JPA 관리 객체를 직접 변경 | 도메인 객체를 변경하고 변환 후 저장 |
| 도메인 테스트 | JPA 제약을 함께 고려 | JPA 없이 단위 테스트 가능 |
| 영속성 변경 | 도메인과 매핑을 함께 수정 | 엔티티·매퍼에 변경을 국한 |
| JPA 기능 | Dirty Checking·Embeddable 직접 사용 | 영속성 계층 안에서만 사용 |
| 주요 위험 | 기술 규칙과 비즈니스 규칙의 결합 | 필드 불일치·변환 누락 |

통합 모델은 구현량이 적고 Dirty Checking을 바로 쓴다. 분리 모델은 JPA 기본 생성자·프록시·컬럼 변경을 도메인에 침투시키지 않지만 변환과 통합 테스트 비용이 든다. **현재 선택: 변경 이유 분리와 도메인 단위 테스트의 이점이 더 크다.**

## 3. Delivery Aggregate와 VO 예시

Setty delivery의 Delivery Aggregate와 VO 어휘를 예시로 채택한다. 현재 소스는 Delivery와 VO에 JPA 애너테이션을 직접 둔 통합 모델이다. 이는 비교 대상이며, 이 문서의 선택은 아래처럼 분리한 구현이다.

Delivery만 배송 핵심 Aggregate Root다. Order와 기사 계정은 각각 다른 Aggregate이므로 OrderId와 DriverId로만 참조한다. DeliveryRoute, FurnitureInfo, EstimatedDeliveryFee, DeliveryAssignment는 Delivery에 종속된 VO다.

Application → Domain, Persistence → Domain, Domain ↛ Persistence로 의존 방향을 고정한다.

### 3.1 순수 Delivery Aggregate

Delivery는 상태 전이와 담당 기사 검증을 수행한다. 요청 시각과 상태 변경 시각은 외부에서 인자로 받는다. Domain이 현재 시각, HTTP 요청, JPA를 직접 조회하지 않는다.

~~~java
public final class Delivery {

    private final DeliveryId id; // 신규 Aggregate에서는 null 가능
    private final OrderId orderId;
    private final FurnitureInfo furnitureInfo;
    private final DeliveryRoute route;
    private final EstimatedDeliveryFee estimatedDeliveryFee;
    private DeliveryAssignment assignment;
    private DeliveryStatus status;
    private final Instant requestedAt;
    private Instant pickedUpAt;
    private Instant deliveredAt;

    private Delivery(DeliveryState state) {
        id = state.id();
        orderId = requireNonNull(state.orderId());
        furnitureInfo = requireNonNull(state.furnitureInfo());
        route = requireNonNull(state.route());
        estimatedDeliveryFee = requireNonNull(state.estimatedDeliveryFee());
        assignment = state.assignment();
        status = requireNonNull(state.status());
        requestedAt = requireNonNull(state.requestedAt());
        pickedUpAt = state.pickedUpAt();
        deliveredAt = state.deliveredAt();
        validateState();
    }

    public static Delivery request(
            OrderId orderId,
            FurnitureInfo furnitureInfo,
            DeliveryRoute route,
            EstimatedDeliveryFee estimatedDeliveryFee,
            Instant requestedAt
    ) {
        return new Delivery(new DeliveryState(null, orderId, furnitureInfo, route,
                estimatedDeliveryFee, null, DeliveryStatus.REQUESTED, requestedAt, null, null));
    }

    public static Delivery restore(DeliveryState state) {
        return new Delivery(state);
    }

    public void accept(DriverId driverId, Instant acceptedAt) {
        requireStatus(DeliveryStatus.REQUESTED);
        assignment = new DeliveryAssignment(requireNonNull(driverId), requireNonNull(acceptedAt));
        status = DeliveryStatus.ACCEPTED;
    }

    public void pickUp(DriverId driverId, Instant pickedUpAt) {
        requireStatus(DeliveryStatus.ACCEPTED);
        requireAssignedDriver(driverId);
        this.pickedUpAt = requireNonNull(pickedUpAt);
        status = DeliveryStatus.PICKED_UP;
    }

    public void complete(DriverId driverId, Instant deliveredAt) {
        requireStatus(DeliveryStatus.PICKED_UP);
        requireAssignedDriver(driverId);
        this.deliveredAt = requireNonNull(deliveredAt);
        status = DeliveryStatus.DELIVERED;
    }

    private void requireStatus(DeliveryStatus expected) {
        if (status != expected) {
            throw new InvalidDeliveryTransition(status, expected);
        }
    }

    private void requireAssignedDriver(DriverId driverId) {
        if (assignment == null || !assignment.isAssignedTo(driverId)) {
            throw new DeliveryDriverMismatch(driverId);
        }
    }

    // validateState(), 접근자, requireNonNull() 생략
}

public record DeliveryState(
        DeliveryId id, OrderId orderId, FurnitureInfo furnitureInfo, DeliveryRoute route,
        EstimatedDeliveryFee estimatedDeliveryFee, DeliveryAssignment assignment,
        DeliveryStatus status, Instant requestedAt, Instant pickedUpAt, Instant deliveredAt
) {}
~~~

Delivery의 허용 전이는 REQUESTED → ACCEPTED → PICKED_UP → DELIVERED다. 공통 DeliveryStatus에 PENDING이 있더라도 Delivery Aggregate의 시작 상태는 REQUESTED다.

### 3.2 VO로 Aggregate 입력을 제한

원시 Long과 String만 넘기지 않는다. OrderId와 DriverId는 같은 Long이라도 서로 대입할 수 없다. DeliveryRoute는 주소와 연락처의 완전한 조합만 허용한다.

~~~java
public record DeliveryId(Long value) { public DeliveryId { requirePositive(value, "deliveryId"); } }
public record OrderId(Long value) { public OrderId { requirePositive(value, "orderId"); } }
public record DriverId(Long value) { public DriverId { requirePositive(value, "driverId"); } }
public record Address(String value) { public Address { value = requireText(value, "주소"); } }
public record PhoneNumber(String value) { public PhoneNumber { value = requireText(value, "연락처"); } }

public record FurnitureInfo(String itemName, String category) {
    public FurnitureInfo { itemName = requireText(itemName, "가구명"); category = requireText(category, "카테고리"); }
}

public record EstimatedDeliveryFee(Integer value) {
    public EstimatedDeliveryFee { if (value == null || value < 0) throw new IllegalArgumentException("예상 배송비는 0 이상이어야 합니다."); }
}

public record DeliveryRoute(
        Address pickupAddress,
        Address deliveryAddress,
        PhoneNumber pickupPhoneNumber,
        PhoneNumber deliveryPhoneNumber
) {
    public DeliveryRoute {
        requireNonNull(pickupAddress); requireNonNull(deliveryAddress);
        requireNonNull(pickupPhoneNumber); requireNonNull(deliveryPhoneNumber);
    }
}

public record DeliveryAssignment(DriverId driverId, Instant acceptedAt) {
    public DeliveryAssignment { requireNonNull(driverId); requireNonNull(acceptedAt); }
    public boolean isAssignedTo(DriverId candidate) { return driverId.equals(candidate); }
}
~~~

**판단: VO는 자체 입력 불변식과 동등성만 책임진다.** 상태 전이와 담당 기사 권한처럼 여러 값의 일관성을 바꾸는 규칙은 Delivery Aggregate에 둔다.

### 3.3 JPA 엔티티와 변환

DeliveryEntity는 DB 구조와 JPA 동작만 표현한다. Order는 다른 Aggregate이므로 Long orderId로 저장하고 Entity 참조를 만들지 않는다. DeliveryRoute와 DeliveryAssignment처럼 여러 컬럼을 함께 저장하는 값만 영속성 전용 Embeddable로 둔다.

~~~java
@Entity
@Table(name = "delivery")
@NoArgsConstructor(access = AccessLevel.PROTECTED)
public class DeliveryEntity {

    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    @Column(name = "order_id", nullable = false, unique = true)
    private Long orderId;
    @Embedded private FurnitureInfoEmbeddable furnitureInfo;
    @Embedded private DeliveryRouteEmbeddable route;
    @Column(name = "estimated_fee", nullable = false)
    private Integer estimatedDeliveryFee;
    @Embedded private DeliveryAssignmentEmbeddable assignment;
    @Column(nullable = false, length = 20)
    private String status;
    @Column(name = "requested_at", nullable = false, updatable = false)
    private Instant requestedAt;
    @Column(name = "picked_up_at") private Instant pickedUpAt;
    @Column(name = "delivered_at") private Instant deliveredAt;

    public static DeliveryEntity from(Delivery delivery) {
        return new DeliveryEntity(
                delivery.id() == null ? null : delivery.id().value(), delivery.orderId().value(),
                FurnitureInfoEmbeddable.from(delivery.furnitureInfo()), DeliveryRouteEmbeddable.from(delivery.route()),
                delivery.estimatedDeliveryFee().value(), DeliveryAssignmentEmbeddable.from(delivery.assignment()),
                delivery.status().name(), delivery.requestedAt(), delivery.pickedUpAt(), delivery.deliveredAt());
    }

    public Delivery toDomain() {
        return Delivery.restore(new DeliveryState(
                id == null ? null : new DeliveryId(id), new OrderId(orderId),
                furnitureInfo.toDomain(), route.toDomain(), new EstimatedDeliveryFee(estimatedDeliveryFee),
                assignment == null ? null : assignment.toDomain(), toDomainStatus(status),
                requestedAt, pickedUpAt, deliveredAt));
    }

    private static DeliveryStatus toDomainStatus(String storedStatus) {
        try {
            return DeliveryStatus.valueOf(storedStatus);
        } catch (IllegalArgumentException exception) {
            throw new IllegalStateException("저장된 배송 상태가 유효하지 않습니다: " + storedStatus, exception);
        }
    }

    // private 생성자 생략
}
~~~

| 도메인 값 | 영속성 표현 | 변환 책임 |
| --- | --- | --- |
| DeliveryId | Long id | value() / new DeliveryId(id) |
| OrderId | Long orderId | value() / new OrderId(orderId) |
| FurnitureInfo | FurnitureInfoEmbeddable | from() / toDomain() |
| DeliveryRoute | DeliveryRouteEmbeddable | from() / toDomain() |
| EstimatedDeliveryFee | Integer estimatedDeliveryFee | value() / 생성자 |
| DeliveryAssignment | DeliveryAssignmentEmbeddable | from() / toDomain() |
| DeliveryStatus | String status | name() / valueOf() |

### 3.4 영속성 전용 Embeddable

DeliveryRouteEmbeddable과 DeliveryAssignmentEmbeddable은 JPA에만 존재한다. 도메인 VO와 같은 데이터를 담아도 DB 컬럼명·기본 생성자·null 복원 규칙이라는 다른 변경 이유를 가진다.

~~~java
@Embeddable
@NoArgsConstructor(access = AccessLevel.PROTECTED)
public class DeliveryRouteEmbeddable {

    @Column(name = "pickup_address", nullable = false) private String pickupAddress;
    @Column(name = "delivery_address", nullable = false) private String deliveryAddress;
    @Column(name = "pickup_phone_number", nullable = false) private String pickupPhoneNumber;
    @Column(name = "delivery_phone_number", nullable = false) private String deliveryPhoneNumber;

    public static DeliveryRouteEmbeddable from(DeliveryRoute route) {
        return new DeliveryRouteEmbeddable(route.pickupAddress().value(), route.deliveryAddress().value(),
                route.pickupPhoneNumber().value(), route.deliveryPhoneNumber().value());
    }

    public DeliveryRoute toDomain() {
        return new DeliveryRoute(new Address(pickupAddress), new Address(deliveryAddress),
                new PhoneNumber(pickupPhoneNumber), new PhoneNumber(deliveryPhoneNumber));
    }

    // private 생성자 생략
}

@Embeddable
@NoArgsConstructor(access = AccessLevel.PROTECTED)
public class DeliveryAssignmentEmbeddable {

    @Column(name = "driver_id") private Long driverId;
    @Column(name = "accepted_at") private Instant acceptedAt;

    public static DeliveryAssignmentEmbeddable from(DeliveryAssignment assignment) {
        return assignment == null ? null : new DeliveryAssignmentEmbeddable(
                assignment.driverId().value(), assignment.acceptedAt());
    }

    public DeliveryAssignment toDomain() {
        return new DeliveryAssignment(new DriverId(driverId), acceptedAt);
    }

    // private 생성자 생략
}
~~~

FurnitureInfoEmbeddable도 같은 방식으로 itemName과 category를 저장한다. assignment의 두 컬럼이 모두 null이면 미배정 상태다. 하나만 null이면 유효한 DeliveryAssignment로 복원할 수 없으므로 오류로 처리하고 DB 제약을 검증한다.

### 3.5 Application과 저장 어댑터

Application은 경계 입력을 VO로 만들고 Aggregate의 팩터리·행위를 호출한다. Repository는 Aggregate 전체를 저장한다. 개별 VO용 Repository를 만들지 않는다.

~~~java
@Transactional
public Delivery register(RegisterDeliveryCommand command, Instant requestedAt) {
    Delivery delivery = Delivery.request(
            new OrderId(command.orderId()),
            new FurnitureInfo(command.itemName(), command.category()),
            new DeliveryRoute(
                    new Address(command.pickupAddress()),
                    new Address(command.deliveryAddress()),
                    new PhoneNumber(command.pickupPhoneNumber()),
                    new PhoneNumber(command.deliveryPhoneNumber())
            ),
            new EstimatedDeliveryFee(command.estimatedDeliveryFee()),
            requestedAt
    );
    return deliveryRepository.save(delivery);
}

public final class JpaDeliveryRepository implements DeliveryRepository {

    private final SpringDataDeliveryJpaRepository repository;

    @Override
    public Delivery save(Delivery delivery) {
        return repository.save(DeliveryEntity.from(delivery)).toDomain();
    }
}
~~~

저장 흐름은 요청 DTO → Command → VO 생성 → Delivery.request() 또는 도메인 행위 → Entity.from() → JpaRepository.save() → toDomain()이다. 저장 후에는 ID가 채워진 새 Delivery를 반환한다.

**판단: Delivery가 Aggregate 저장 단위다.** OrderId, DeliveryRoute, DeliveryAssignment를 독립적으로 저장하거나 다른 Aggregate Entity를 탐색하지 않는다.

## 4. Enumerated 판단

Jakarta Persistence의 enum 기본 저장 방식은 ORDINAL이다. STRING은 enum 상수 이름을 저장한다.

| 상황 | 판단 |
| --- | --- |
| enum 상수명이 DB의 공식 저장값 | 엔티티 필드에서 Enumerated(EnumType.STRING) 허용 |
| DB 코드가 enum 이름과 다름 | 엔티티의 String과 명시적 코드 변환 |
| enum 이름 변경·값 제거 | DB 마이그레이션과 이전 값 호환 변환 |
| Enumerated 옵션 생략 | 거부. 기본 ORDINAL에 의존하지 않음 |
| Enumerated(EnumType.ORDINAL) | 거부. 상수 순서 변경이 저장 의미를 바꿀 수 있음 |

현재 Delivery 소스는 DeliveryStatus에 Enumerated(EnumType.STRING)을 적용한다. 분리 구조에서도 상태 이름이 DB 계약으로 확정됐다면 같은 선택을 할 수 있다.

~~~java
@Enumerated(EnumType.STRING)
@Column(nullable = false, length = 20)
private DeliveryStatus status;
~~~

상태 이름이 내부 구현 세부 사항이거나 외부 코드로 관리되면 위 예시 대신 String status와 명시적 변환을 사용한다. 이 문서의 DeliveryEntity 예시는 후자를 기본값으로 둔다.

## 5. Embeddable 판단

Embeddable은 독립 영속 식별자 없이 소유 엔티티의 일부로 저장되는 복합 값 매핑이다. 도메인 VO와 자주 대응하지만 같은 개념은 아니다.

| 상황 | 판단 |
| --- | --- |
| 한 컬럼이면 충분함. 예: DeliveryStatus, DeliveryId, OrderId | Embeddable 거부. 기본 컬럼과 변환 사용 |
| 여러 컬럼이 함께 움직임. 예: DeliveryRoute, FurnitureInfo, DeliveryAssignment | 영속성 전용 Embeddable 검토 |
| 독립 ID·Repository·상태 전이가 필요함 | Embeddable 거부. 엔티티 검토 |
| 순수 도메인 타입에 Embeddable을 붙여야 함 | 현재 선택에서는 거부 |

DeliveryRoute는 네 값이 함께 있어야만 유효하고 DeliveryAssignment는 DriverId와 수락 시각이 함께 있어야 한다. 따라서 영속성에서는 각각 하나의 Embeddable로 평탄화할 수 있다. 도메인에서는 JPA 애너테이션 없이 동일한 불변식을 유지한다.

## 6. 검증 방법

### 도메인 단위 테스트

- JPA·Spring 없이 Delivery.request()와 상태 전이를 검증한다.
- REQUESTED → ACCEPTED → PICKED_UP → DELIVERED만 허용되는지 확인한다.
- 재수락, 건너뛴 상태 전이, 다른 기사의 pickUp·complete를 거부하는지 확인한다.
- 양수 ID, 공백 제거된 주소·연락처·가구 정보, 음수 예상 배송비 거부를 검증한다.
- DeliveryRoute의 구성 요소 누락과 DeliveryAssignment의 driverId·acceptedAt 누락을 검증한다.

### 의존성과 Repository 통합 테스트

- domain의 jakarta.persistence, Hibernate, Spring Data import가 0건인지 자동 검사한다.
- Delivery → DeliveryEntity → Delivery 왕복 변환 뒤 상태·VO·시각이 보존되는지 확인한다.
- 저장 뒤 생성 ID가 DeliveryId로 복원되는지 확인한다.
- order_id가 다른 Aggregate의 ID 값으로 저장되는지 확인한다.
- 상태 저장값, 알 수 없는 과거 상태 값, DeliveryAssignmentEmbeddable의 전체·부분 null을 검증한다.

## 최종 판단 기준

> 도메인 모델은 비즈니스 규칙과 상태 전이를 책임지고 JPA를 모른다. Delivery 같은 Aggregate는 상태 전이와 여러 VO의 일관성을 소유하며, 다른 Aggregate는 ID VO로만 참조한다. JPA 엔티티는 해당 ID를 원시 컬럼으로 저장하고 도메인 모델을 변환한다. enum은 기본 ORDINAL을 사용하지 않고 저장 문자열을 명시하며, Embeddable은 영속성 전용 복합 값 매핑에만 사용한다.

## 근거

- Delivery, DeliveryRoute, DeliveryAssignment, OrderId, DriverId 등 배송 도메인의 실제 코드 어휘와 상태 전이를 예시로 채택했다. 원본 저장소 링크는 기록하지 않는다.
- 현재 Delivery 소스는 통합 모델이므로, 분리 모델의 구현 선택과 구분해서 기록했다.
- enum·embeddable의 영속성 의미는 Jakarta Persistence 명세로 확인했다. 외부 링크는 기록하지 않는다.

## 검증 범위

- 확인됨: Delivery Aggregate의 상태 전이, VO 불변식, Order·Driver ID 참조, enum·embeddable의 영속성 의미
- 확인 필요: 실제 프로젝트 적용 후 변환 코드량, 조회 성능, 변경 비용
- 비범위: JPA를 사용하지 않는 순수 계산 라이브러리의 객체 분류

## 후속 검증

- [ ] Delivery 쓰기 유스케이스 하나에 Domain → Entity → Domain 변환을 적용한다.
- [ ] domain 패키지의 JPA·Hibernate·Spring Data import 0건을 자동 검사한다.
- [ ] DeliveryStatus 이름 변경과 저장값 변경을 포함한 DB 마이그레이션을 재현한다.
- [ ] DeliveryRoute를 평탄화 방식과 영속성 전용 Embeddable 방식으로 각각 구현해 변경 비용을 비교한다.
