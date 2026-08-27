# Decision Criteria

개인 판단 기준을 문서 단위로 기록하고 수정하는 저장소.

## 운영 규칙

1. 문서 하나에는 하나의 판단 주제만 기록한다.
2. 판단은 결론, 근거, 반례·제약, 검증 상태를 함께 남긴다.
3. 사실로 확인한 내용과 원인에 대한 추측을 구분한다.
4. 실제 사례가 기준과 충돌하면 기존 문서를 수정한다.
5. 판단 기준 문서는 `main`에 직접 커밋한다. PR은 협업이나 별도 검토가 필요할 때만 사용한다.
6. 커밋 하나에는 하나의 판단 주제만 포함한다.

## 기준 목록

- [JPA 연관관계와 조회 계획](criteria/jpa-association-fetch-plan.md)
- [추상 클래스와 인터페이스 선택](criteria/abstract-class-vs-interface.md)
- [테스트 전략과 계층별 책임](criteria/testing-strategy.md)

## 디렉터리

- `criteria/`: 확정 또는 검토 중인 판단 기준
- `.github/pull_request_template.md`: 별도 검토가 필요한 변경을 위한 선택적 PR 템플릿

## 문서 상태

- `draft`: 초안. 추가 사례 또는 검증 필요
- `validated`: 실제 사례와 근거로 검증됨
- `revised`: 기존 판단을 수정함
