# 도메인 모델과 JPA 엔티티 분리

- 상태: draft
- 예시 기준: Delivery → DeliveryEntity
- 주제: 순수 도메인 모델, JPA 엔티티, 변환 책임
- 선택: 도메인 모델과 JPA 엔티티 분리

## 결론

**채택: 비즈니스 규칙을 가진 도메인 모델과 DB 매핑을 위한 JPA 엔티티를 별도 클래스로 둔다.** 도메인 모델은 JPA·Hibernate·Spring Data 애너테이션과 타입을 import하지 않는다.

**채택: 영속성 계층이 변환을 소유한다.** DeliveryEntity.from(delivery)과 toDomain()은 영속성 계층에 둔다. 데이터는 양방향으로 이동해도 코드 의존성은 Persistence → Domain 단방향으로 유지한다.

**채택: 다른 Aggregate는 두 계층 모두 ID로 참조한다.** 도메인은 의미가 구분되는 ID VO를, JPA 엔티티는 원시 ID 컬럼을 사용한다.

## 빠른 기준

| 질문 | 판단 |
| --- | --- |
| JPA 없이도 생성·변경·검증돼야 하는 비즈니스 규칙인가? | 순수 도메인 모델 |
| 테이블·컬럼·기본 생성자·영속화 표현을 다루는가? | JPA 엔티티 또는 영속성 매퍼 |
| 다른 Aggregate를 참조하는가? | 도메인과 JPA 엔티티 모두 ID로 참조 |
| 도메인 객체를 DB에 저장·복원하는가? | 영속성 계층이 변환 |
| 값의 DB 저장 형태를 결정하는가? | 별도 기준 참조 |

## 1. 적용 범위

이 기준은 도메인 모델과 JPA 엔티티의 분리, 의존 방향, 변환 책임을 다룬다.

포함:

- 비즈니스 규칙과 DB 매핑의 클래스 분리
- Persistence → Domain 의존 방향
- from()·toDomain()의 소유자
- 다른 Aggregate의 ID 참조

비범위:

- Aggregate 상태 전이와 VO 불변식
- DB 값 저장 방식
- 조회 모델과 조회 계획

값 저장 방식의 상세 판단은 [JPA 값 저장 방식](jpa-value-storage.md)에서 다룬다.

## 2. 두 방식의 트레이드오프

| 항목 | 통합 모델: 도메인 = JPA 엔티티 | 분리 모델: 도메인 ≠ JPA 엔티티 — 현재 선택 |
| --- | --- | --- |
| 클래스 수 | 적음 | 도메인·엔티티·변환 코드 증가 |
| 도메인 테스트 | JPA 제약을 함께 고려 | JPA 없이 단위 테스트 가능 |
| 영속성 변경 | 도메인과 매핑을 함께 수정 | 엔티티·매퍼에 변경을 국한 |
| JPA 기능 | 도메인 타입에 직접 적용 | 영속성 계층 안에서만 사용 |
| 주요 위험 | 기술 규칙과 비즈니스 규칙의 결합 | 필드 불일치·변환 누락 |

통합 모델은 구현량이 적고 JPA 기능을 바로 쓴다. 분리 모델은 DB 구조 변경이 도메인 모델을 직접 바꾸지 못하게 하지만 변환 코드와 통합 테스트 비용이 든다. **현재 선택: 변경 이유 분리와 도메인 단위 테스트의 이점이 더 크다.**

## 3. Delivery 적용

Delivery는 도메인 모델이고 DeliveryEntity는 영속성 모델이다. Delivery의 상태 전이와 VO 세부 규칙은 이 문서에서 판단하지 않는다.

~~~text
Application → Domain
Persistence → Domain
Domain ↛ Persistence

Delivery
└── OrderId

DeliveryEntity
└── Long orderId
~~~

도메인은 toEntity()를 제공하지 않는다. 저장 어댑터가 DeliveryEntity.from(delivery)으로 저장 표현을 만들고, 저장 결과를 toDomain()으로 복원한다.

**판단: 변환은 기술 경계에서 끝낸다.** Application과 도메인 행위는 JPA 엔티티를 직접 받거나 반환하지 않는다.

## 4. 검증 방법

- domain 패키지의 jakarta.persistence, Hibernate, Spring Data import가 0건인지 검사한다.
- Application이 JPA Entity가 아닌 도메인 모델을 입력·반환하는지 확인한다.
- 변환 함수가 영속성 패키지에만 있는지 확인한다.
- 저장 표현의 왕복 변환과 컬럼 제약은 [JPA 값 저장 방식](jpa-value-storage.md)의 검증 항목으로 확인한다.

## 최종 판단 기준

> 도메인 모델은 비즈니스 규칙을 책임지고 JPA를 모른다. JPA 엔티티는 저장 표현을 책임지며 영속성 계층이 두 모델을 변환한다. 다른 Aggregate는 두 계층 모두 ID로 참조한다.

## 검증 범위

- 확인 필요: 실제 적용 후 변환 코드량과 변경 비용
- 비범위: Aggregate·VO 세부 설계와 DB 값 저장 방식
