# Decision Criteria

개인 판단 기준을 문서 단위로 기록하고 수정하는 저장소.

## 운영 규칙

1. 문서 하나에는 하나의 판단 주제만 기록한다.
2. 판단은 결론, 근거, 반례·제약, 검증 상태를 함께 남긴다.
3. 특정 저장소와 PR은 판단 기준을 설명하는 예시로만 사용한다. 판단의 근거나 검증 상태로 사용하지 않는다.
4. 사실로 확인한 내용과 원인에 대한 추측을 구분한다.
5. 실제 사례가 기준과 충돌하면 기존 문서를 수정한다.
6. 판단 기준 문서는 `main`에 직접 커밋한다. PR은 협업이나 별도 검토가 필요할 때만 사용한다.
7. 커밋 하나에는 하나의 판단 주제만 포함한다.

## 기준 목록

| 분야 | 판단 기준 |
| --- | --- |
| 도메인·영속성 | [JPA 엔티티의 Aggregate 참조 방식](criteria/jpa-aggregate-reference.md) |
| 도메인·영속성 | [JPA 조회 계획과 N+1](criteria/jpa-fetch-plan.md) |
| 도메인·영속성 | [도메인 모델과 JPA 엔티티 분리](criteria/domain-model-and-jpa-entity.md) |
| 도메인·영속성 | [JPA 값 저장 방식](criteria/jpa-value-storage.md) |
| 도메인·영속성 | [SQL과 Service 로직의 구분](criteria/sql-vs-service-logic.md) |
| 객체 설계·Java | [추상 클래스와 인터페이스 선택](criteria/abstract-class-vs-interface.md) |
| 객체 설계·Java | [Java `final` 키워드 사용](criteria/java-final-keyword.md) |
| 테스트·검증 | [테스트 전략과 계층별 책임](criteria/testing-strategy.md) |
| 테스트·검증 | [검증 책임과 DB 제약](criteria/validation-and-db-constraints.md) |
| 인증·인가 | [세션과 토큰 기반 인증 상태 선택](criteria/session-vs-token.md) |
| 인증·인가 | [Cookie와 Authorization 헤더의 인증 정보 전달 선택](criteria/cookie-vs-authorization-header.md) |
| 애플리케이션 통신 | [비동기 메시지 전달 방식 선택](criteria/async-message-delivery.md) |
| 애플리케이션 통신 | [SSE를 통한 목록 변경 알림](criteria/sse-list-change-notification.md) |

## 디렉터리

- `criteria/`: 확정 또는 검토 중인 판단 기준
- `.github/pull_request_template.md`: 별도 검토가 필요한 변경을 위한 선택적 PR 템플릿

## 문서 상태

- `draft`: 결론이나 선택 기준이 아직 정리되지 않음
- `adopted`: 판단 기준과 트레이드오프가 확정됨
- `validated`: 독립적인 실험·측정·명세로 기술적 전제를 확인함
- `revised`: 기존 판단을 변경함

상태는 저장소 예시의 유무로 결정하지 않는다.
