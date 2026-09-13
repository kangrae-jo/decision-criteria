# DB 변경과 이벤트 발행 일관성

- 상태: `adopted`
- 검토일: 2026-09-13
- 주제: Dual Write, CDC, Transactional Outbox, Event Sourcing
- 선택: 원본 데이터와 이벤트 계약에 따라 발행 방식 결정

## 결론

**거부: 유실되면 안 되는 이벤트를 DB 저장과 브로커 발행의 직접 Dual Write로 처리한다.** 두 시스템을 하나의 로컬 트랜잭션으로 묶을 수 없으면 한쪽만 성공하는 상태가 남는다.

**채택: 현재 상태 DB가 원본이고 애플리케이션이 비즈니스 이벤트 계약을 소유하면 Transactional Outbox를 우선 검토한다.**

**조건부 채택: 기존 DB의 행 변경을 낮은 애플리케이션 침투로 전달하고 물리 스키마 기반 이벤트를 허용하면 CDC를 검토한다.**

**조건부 채택: 전체 변경 이력과 상태 재구성이 도메인의 핵심 요구사항이면 Event Sourcing을 검토한다.** 신뢰할 수 있는 이벤트 발행만을 위해 도입하지 않는다.

**거부: 진실은 언제나 특정 DB나 브로커에 있다고 일반화한다.** 현재 상태 DB, 이벤트 스토어, 외부 로그 중 무엇이 원본인지는 아키텍처가 명시해야 한다.

## 빠른 기준

| 상황 | 우선 검토 |
| --- | --- |
| 이벤트 누락과 불일치를 허용 | 트랜잭션 완료 후 직접 발행 |
| DB 행 변경 자체를 전달 | CDC |
| 비즈니스 이벤트 계약을 직접 설계 | Transactional Outbox |
| 현재 상태 DB가 원본 | CDC 또는 Outbox |
| 이벤트 이력이 원본이며 상태를 재구성 | Event Sourcing |
| 원본 DB와 Outbox를 같은 트랜잭션에 저장할 수 없음 | 경계 재설계 또는 별도 조정 방식 |

## 1. 직접 Dual Write

```text
DB 저장
→ 브로커 발행
```

두 작업 사이에는 다음 실패가 존재한다.

| 결과 | 문제 |
| --- | --- |
| DB 성공, 발행 실패 | 저장됐지만 소비자가 변경을 모름 |
| 발행 성공, DB 실패 | 존재하지 않는 상태를 소비자가 처리 |
| 발행 성공, 응답 유실 | 재시도로 중복 이벤트 발생 가능 |

**거부 조건:** 검색 색인, 결제, 재고, 다른 서비스 상태처럼 이벤트 누락이 데이터 불일치로 이어지는 경우.

**허용 조건:** 알림 누락을 허용하고 최신 상태를 원본 DB에서 다시 조회할 수 있는 경우. 이때도 직접 발행은 전달 보장이 아닌 최선 노력으로 기록한다.

## 2. CDC

CDC는 DB의 binlog, WAL 같은 트랜잭션 로그에서 커밋된 행 변경을 읽어 전달한다.

### 채택 조건

- 기존 애플리케이션의 쓰기 경로 변경을 최소화해야 한다.
- INSERT, UPDATE, DELETE라는 물리적 변경이 소비자에게 필요한 계약이다.
- DB 복제 로그와 커넥터를 운영할 수 있다.
- 스키마 변경을 소비자와 조정할 수 있다.

### 비용·제약

- 기본 변경 이벤트가 테이블과 컬럼 구조를 반영한다.
- 비즈니스 의미를 소비자나 변환 계층이 해석해야 한다.
- replication slot, binlog·WAL 보존, connector lag를 운영해야 한다.
- 스키마 변경과 초기 snapshot이 소비자 계약에 영향을 줄 수 있다.

**판단: CDC가 애플리케이션 코드 침투를 줄일 수는 있지만 전체 도입 비용이 항상 가장 낮은 것은 아니다.**

## 3. Transactional Outbox

현재 상태와 발행할 이벤트를 같은 DB 트랜잭션에 저장한다.

```text
BEGIN
  orders 변경
  outbox_event 추가
COMMIT

Relay
  outbox_event 읽기
  브로커 발행
```

### 채택 조건

- 현재 상태 DB가 원본이다.
- `OrderConfirmed`처럼 비즈니스 의미가 있는 이벤트 계약이 필요하다.
- DB 스키마와 외부 이벤트 스키마를 분리해야 한다.
- DB 변경과 발행 의도의 원자성을 보장해야 한다.

### 보장 범위

- 도메인 상태와 Outbox 레코드는 함께 커밋되거나 함께 롤백된다.
- 브로커 발행은 별도 Relay가 수행하므로 즉시 완료되지 않는다.
- Relay 장애 후 재시도 과정에서 같은 이벤트가 중복 발행될 수 있다.
- 소비자는 eventId를 기준으로 멱등 처리해야 한다.

Outbox Relay는 polling으로 구현하거나 CDC로 Outbox 테이블을 읽을 수 있다. Debezium은 선택지이며 Outbox의 필수 구성요소가 아니다.

### 운영 비용

- 모든 대상 트랜잭션에 Outbox INSERT 추가
- Relay 상태와 발행 지연 관찰
- 미발행 이벤트 재시도
- 처리 완료 레코드의 보존·삭제·파티셔닝
- Aggregate 단위 발행 순서 관리

## 4. Event Sourcing

Event Sourcing은 현재 상태 대신 변경 이벤트를 append-only 이벤트 스토어에 저장한다.

```text
OrderCreated
→ OrderPaid
→ OrderConfirmed
→ 현재 상태 재구성
```

### 채택 조건

- 무엇이 왜 변경됐는지가 핵심 비즈니스 데이터다.
- 과거 시점 상태 재구성과 감사 추적이 필수다.
- 이벤트가 시스템의 권위 있는 원본이어야 한다.
- 이벤트 버전 관리와 projection 재구축 비용을 감당할 수 있다.

### 비용·제약

- 현재 상태 조회를 위한 projection이나 snapshot이 필요할 수 있다.
- 과거 이벤트를 삭제·수정하기 어렵다.
- 이벤트 스키마 버전과 upcasting 전략이 필요하다.
- 읽기 모델은 이벤트 스토어보다 늦게 반영될 수 있다.
- Event Sourcing과 CQRS는 자주 함께 사용하지만 같은 패턴은 아니며 항상 함께 도입할 필요는 없다.

**판단: 이벤트 스토어 append 성공은 원본 이벤트 저장의 성공이다. 외부 브로커, 검색 색인, 알림까지 전달됐다는 뜻은 아니다.** 외부 전달에는 subscription checkpoint, 재시도, 멱등 처리가 별도로 필요하다.

## 5. 원본 데이터 위치

| 방식 | 권위 있는 원본 | 브로커의 역할 |
| --- | --- | --- |
| CDC | 현재 상태 DB | 행 변경 전달 |
| Outbox | 현재 상태 DB | 비즈니스 이벤트 전달 |
| Event Sourcing | 이벤트 스토어 | 구독자와 외부 시스템으로 전달 |

Kafka를 사용한다고 자동으로 이벤트가 원본이 되지 않는다. 반대로 Event Sourcing의 이벤트 스토어는 관계형 DB일 필요가 없다.

## 6. 공통 이벤트 계약

중복, 순서, 스키마 변경을 처리하려면 최소한 다음 값을 명시한다.

| 필드 | 목적 |
| --- | --- |
| eventId | 중복 처리 방지 |
| eventType | 비즈니스 의미 구분 |
| aggregateId | 같은 Aggregate의 순서와 라우팅 |
| aggregateVersion 또는 sequence | 순서 역전 탐지 |
| occurredAt | 발생 시각 |
| schemaVersion | 이벤트 스키마 진화 |
| payload | 소비자에게 필요한 최소 데이터 |

내부 테이블 전체를 그대로 외부 이벤트 계약으로 노출하지 않는다.

## 트레이드오프

| 방식 | 얻는 것 | 부담하는 것 |
| --- | --- | --- |
| 직접 Dual Write | 가장 단순한 구현 | 유실·유령 이벤트·중복 |
| CDC | 쓰기 코드 변경 감소, 커밋 순서 활용 | 물리 스키마 결합, connector 운영 |
| Outbox | 상태와 발행 의도 원자성, 명시적 이벤트 | 테이블·Relay·중복·정리 비용 |
| Event Sourcing | 전체 이력, 상태 재구성, 감사 | 데이터 모델 전환과 가장 큰 운영 복잡도 |

복잡도 순서는 시스템과 기존 운영 환경에 따라 달라진다. CDC가 항상 Outbox보다 싸고 Event Sourcing이 항상 필요한 최종 단계라는 식으로 보지 않는다.

## 검증 방법

| 실패 지점 | 확인할 결과 |
| --- | --- |
| 도메인 저장 전 예외 | 상태와 이벤트 모두 없음 |
| Outbox 저장 전 예외 | 전체 DB 트랜잭션 롤백 |
| 커밋 후 Relay 중단 | 재시작 후 미발행 이벤트 전송 |
| 브로커 발행 후 Relay 상태 저장 실패 | 중복 발행을 소비자가 멱등 처리 |
| 같은 Aggregate 이벤트 병렬 처리 | sequence로 순서 역전 탐지 |
| 이벤트 스키마 변경 | 이전 버전 소비와 재처리 유지 |
| Outbox 정리 작업 | 미발행 레코드 보존과 DB 부하 확인 |
| CDC 중단 | WAL·binlog 보존 한도 안에서 재개 |

## 기술 문서

- [AWS Transactional Outbox Pattern](https://docs.aws.amazon.com/prescriptive-guidance/latest/cloud-design-patterns/transactional-outbox.html)
- [Debezium Outbox Event Router](https://debezium.io/documentation/reference/transformations/outbox-event-router.html)
- [Debezium MySQL Connector](https://debezium.io/documentation/reference/stable/connectors/mysql.html)
- [Debezium PostgreSQL Connector](https://debezium.io/documentation/reference/stable/connectors/postgresql.html)
- [Azure Event Sourcing Pattern](https://learn.microsoft.com/en-us/azure/architecture/patterns/event-sourcing)
- [Azure CQRS Pattern](https://learn.microsoft.com/en-us/azure/architecture/patterns/cqrs)

## 최종 판단 기준

> 현재 상태 DB가 원본이고 비즈니스 이벤트 계약이 필요하면 Transactional Outbox를 우선 검토한다. 물리적 행 변경 전달로 충분하고 애플리케이션 변경을 줄여야 하면 CDC를 검토한다. 전체 이벤트 이력이 원본이어야 할 때만 Event Sourcing을 선택한다. 신뢰할 수 있는 이벤트를 DB와 브로커에 직접 이중 쓰기하지 않으며, 전달 보장이 필요한 비동기 처리는 중복과 재시도를 전제로 설계한다.
