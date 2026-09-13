# 메시지 브로커 제품 선택

- 상태: `adopted`
- 검토일: 2026-09-13
- 주제: Kafka, RabbitMQ Queue·Stream, Redis Pub/Sub·Streams
- 선택: 전달 모델과 운영 제약을 먼저 정한 뒤 제품 선택

## 결론

**거부: Kafka는 이벤트 브로커, RabbitMQ와 Redis는 메시지 브로커라고 제품 단위로 고정 분류한다.** RabbitMQ와 Redis도 Queue, Stream, Pub/Sub에 따라 보존과 소비 방식이 달라진다.

**채택: 작업 분배와 ACK·재시도가 중심이면 RabbitMQ Queue를 우선 검토한다.**

**채택: 장기간 보존, 재처리, 여러 독립 소비자와 partition 단위 순서가 중심이면 Kafka를 우선 검토한다.**

**채택: 이미 RabbitMQ를 운영하며 제한된 기간의 재생이 필요하면 RabbitMQ Streams를 검토한다.**

**채택: 이미 Redis를 운영하고 짧은 보존 범위의 스트림이 필요하면 Redis Streams를 검토한다.** 장애 시 데이터 안전성과 메모리 사용 조건을 별도로 확인한다.

**채택: 누락을 허용하는 실시간 알림이면 Redis Pub/Sub를 검토한다.**

## 선택 전제

제품을 비교하기 전에 [비동기 메시지 전달 방식 선택](async-message-delivery.md)에 따라 전달 모델을 결정한다.

```text
요구사항
→ 동기 호출 / 일시적 알림 / 작업 큐 / 이벤트 스트림
→ 보존·재처리·순서·소비자·운영 조건
→ 제품 선택
```

## 빠른 기준

| 우선 요구사항 | 우선 검토 |
| --- | --- |
| 작업자 중 한 명의 처리, ACK, 재전달 | RabbitMQ Queue |
| 누락 허용 실시간 알림 | Redis Pub/Sub |
| Redis 안에서 보존과 consumer group 필요 | Redis Streams |
| RabbitMQ 안에서 보존과 반복 읽기 필요 | RabbitMQ Streams |
| 긴 보존, 높은 처리량, 여러 독립 consumer group | Kafka |
| 운영 인력이 부족하고 기존 제품으로 충족 가능 | 기존 운영 제품의 Queue·Stream 기능 |

## 1. RabbitMQ Queue

RabbitMQ Queue는 처리할 작업을 소비자에게 전달하는 경우에 우선 검토한다.

- 소비자가 처리 완료 후 ACK한다.
- ACK하지 않은 메시지는 연결 실패 시 재전달될 수 있다.
- 성공한 메시지를 장기간 반복해서 읽는 것이 기본 목적은 아니다.
- 소비자는 재전달과 중복 처리를 견뎌야 한다.

**제약:** 과거 기록을 신규 소비자가 임의의 시점부터 재생해야 한다면 Queue보다 Stream이 적합하다.

## 2. RabbitMQ Streams

RabbitMQ Streams는 RabbitMQ가 제공하는 별도의 append-only log다.

- 메시지를 소비해도 즉시 제거하지 않는다.
- 보존 기간이나 크기를 기준으로 오래된 데이터를 제거한다.
- 여러 소비자가 같은 데이터를 반복해서 읽을 수 있다.
- 기존 RabbitMQ 운영 경험을 활용할 수 있다.

**제약:** Queue와 소비 API·보존·성능 특성이 다르므로 RabbitMQ를 사용한다는 이유만으로 전환 비용이 사라지지는 않는다.

## 3. Kafka

Kafka는 보존되는 partition 기반 이벤트 스트림이 필요한 경우에 우선 검토한다.

- 소비와 데이터 보존이 분리된다.
- consumer group별로 처리 위치를 관리할 수 있다.
- 같은 key의 이벤트를 같은 partition에 배치해 순서를 관리할 수 있다.
- 장애 복구와 신규 소비자의 과거 이벤트 재처리에 적합하다.

**제약:** partition 수, key, offset, consumer 재조정, 보존 용량을 운영해야 한다. 작업별 우선순위와 지연 실행, 개별 메시지 재시도가 핵심이면 별도 설계가 늘어난다.

## 4. Redis Pub/Sub

Redis Pub/Sub는 연결된 구독자에게 즉시 알리는 경우에 사용한다.

- 별도 이력과 소비 위치를 저장하지 않는다.
- 구독자가 연결되지 않은 동안 메시지를 복구하지 못한다.
- 최신 상태를 다른 원본에서 다시 조회할 수 있어야 한다.

**제약:** 누락 없는 업무 처리와 장애 이후 재처리에 사용하지 않는다.

## 5. Redis Streams

Redis Streams는 Redis의 append-only log 자료구조다.

- ID를 기준으로 과거 데이터를 읽을 수 있다.
- consumer group과 ACK, pending 항목을 지원한다.
- 길이 또는 ID 기준 trimming으로 보존 범위를 제한할 수 있다.
- 하나의 stream 안에서 작업 분배와 스트림 소비를 모두 구성할 수 있다.

**제약:** Redis의 AOF·RDB와 복제 설정에 따라 장애 시 데이터 손실 범위가 달라진다. 하나의 stream은 Kafka partition처럼 자동으로 여러 Redis 인스턴스에 분할되지 않는다.

## 6. 제품 선택 축

| 축 | 확인할 내용 |
| --- | --- |
| 보존 | 소비 후 제거, 시간·크기 기반 보존, 장기 보존 |
| 재처리 | 메시지 재전달, 임의 offset 재생, 신규 소비자 과거 처리 |
| 소비자 | 경쟁 작업자, 독립 consumer group, fan-out |
| 순서 | 불필요, Aggregate key 단위, partition 단위 |
| 실패 | ACK, 재전달, Dead Letter, 중복 허용 범위 |
| 처리량 | 평균·최대 초당 메시지, 메시지 크기 |
| 운영 | 이미 운영 중인 제품, 클러스터·모니터링 역량 |
| 비용 | 메모리·디스크·네트워크와 관리형 서비스 비용 |

## 트레이드오프

| 선택 | 얻는 것 | 부담하는 것 |
| --- | --- | --- |
| RabbitMQ Queue | 명시적 작업 전달과 ACK | 과거 재생 능력 제한 |
| RabbitMQ Streams | RabbitMQ 기반 보존·재생 | Queue와 다른 운영 모델 |
| Kafka | 장기 보존, 재처리, 독립 소비자 | partition과 클러스터 운영 |
| Redis Pub/Sub | 단순하고 빠른 알림 | 보존과 재처리 없음 |
| Redis Streams | 기존 Redis 기반 스트림 | 메모리와 영속성 설정 제약 |

## 거부 기준

- MSA라는 이유만으로 Kafka를 도입한다.
- 이미 Redis가 있다는 이유만으로 중요한 이벤트를 Redis Pub/Sub로 전달한다.
- RabbitMQ Queue에 장기 감사 이력과 임의 재생을 요구한다.
- 제품의 기본 설정을 확인하지 않고 내구성과 exactly-once 처리를 가정한다.
- 처리량을 측정하지 않고 제품 간 성능 우열을 단정한다.

## 검증 방법

- 예상 메시지 크기와 최대 처리량으로 부하 테스트한다.
- 소비자 중단 시간보다 보존 기간이 긴지 확인한다.
- 브로커와 소비자 장애 후 누락·중복·순서를 확인한다.
- 재처리와 Dead Letter 운영 절차를 실제로 실행한다.
- 디스크·메모리 증가와 보존 정책 동작을 측정한다.

## 기술 문서

- [Apache Kafka Introduction](https://kafka.apache.org/documentation/)
- [RabbitMQ Consumer Acknowledgements](https://www.rabbitmq.com/docs/confirms)
- [RabbitMQ Streams](https://www.rabbitmq.com/docs/streams)
- [Redis Streams](https://redis.io/docs/latest/develop/data-types/streams/)
- [Redis Streaming](https://redis.io/docs/latest/develop/use-cases/streaming/)

## 최종 판단 기준

> 제품명이 아니라 Queue·Stream·Pub/Sub의 소비와 보존 방식을 기준으로 선택한다. 작업 분배는 RabbitMQ Queue, 누락 허용 알림은 Redis Pub/Sub, 기존 Redis나 RabbitMQ 안의 제한된 재생은 각 제품의 Streams, 장기 보존과 여러 독립 소비자의 대용량 재처리는 Kafka를 우선 검토한다. 최종 선택은 실제 처리량과 장애 복구 실험으로 확인한다.
