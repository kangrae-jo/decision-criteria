# 부하 테스트 생성기 선택

- 상태: `adopted`
- 검토일: 2026-10-10
- 주제: k6, Gatling, HTTP 부하, 브라우저 부하, 분석 SDK 검증
- 선택: 브라우저 동작과 API 부하를 함께 검증하면 k6, Java 중심 API 부하만 검증하면 Gatling 우선 검토

## 결론

**채택: 실제 브라우저에서 프런트엔드 SDK 실행과 사용자 동작을 확인하면서 API 부하도 생성해야 하면 k6를 선택한다.** `k6/browser`와 프로토콜 수준 시나리오를 한 도구에서 구성할 수 있다.

**조건부 채택: 서버 API 성능이 유일한 목표이고 Java·Spring 팀이 Maven·Gradle과 Java 테스트 코드를 유지하려면 Gatling을 우선 검토한다.**

**보류: 서버 API 부하 생성 성능만으로 k6와 Gatling의 우열을 결정한다.** 같은 부하 모델과 실행 환경에서 측정하기 전에는 단정하지 않는다.

**거부: 다수의 실제 브라우저를 주 부하로 사용한다.** HTTP 가상 사용자가 대부분의 부하를 만들고, 소수의 브라우저 가상 사용자가 실제 화면과 SDK 동작을 확인한다.

## 빠른 기준

| 판단 기준 | k6 | Gatling | 판단 |
| --- | --- | --- | --- |
| 테스트 작성 언어 | JavaScript·TypeScript | Java·Kotlin·Scala·JavaScript·TypeScript | 팀 언어만으로 결정하지 않음 |
| 실제 브라우저 조작 | Chromium에서 클릭·입력·화면 이동 | HTTP 테스트는 JavaScript와 UI를 실행하지 않음 | 브라우저 동작이 필요하면 k6 |
| 프런트 분석 SDK 검증 | 실제 페이지 JavaScript 실행 가능 | HTTP 호출만으로 SDK가 실행되지 않음 | PostHog 수집 경로 검증은 k6 |
| 서버 API 부하 | 지원 | 지원 | 동일 조건 측정 전 우열 보류 |
| Java·Spring 친숙도 | 별도 JS·TS 코드와 CLI | Java SDK와 Maven·Gradle 사용 가능 | API 전용이면 Gatling 유리 |
| 브라우저와 API 통합 | 한 스크립트에서 시나리오 분리 가능 | 별도 브라우저 자동화 필요 | 관리 도구를 줄이면 k6 |
| 부하 생성 비용 | 브라우저 VU 비용이 크고 HTTP VU는 상대적으로 작음 | 프로토콜 수준 부하 생성 | 다수 HTTP와 소수 브라우저 조합 |
| 결제 승인 자동화 | Sandbox나 결제 대역 필요 | Sandbox나 결제 대역 필요 | 도구 선택과 분리 |
| 실제 처리 성능 | 실행 환경과 시나리오에 좌우됨 | 실행 환경과 시나리오에 좌우됨 | 동일 조건 측정 필요 |

Gatling의 JavaScript·TypeScript SDK는 현재 HTTP 프로토콜 범위다. 브라우저 세션을 기록해 HTTP 시뮬레이션을 만들 수 있다는 것과 실제 브라우저를 실행한다는 것은 다르다.

## 1. 테스트 대상을 먼저 분리

부하 생성기의 핵심 판단 기준은 작성 언어가 아니라 검증 대상이다.

- 프로토콜 테스트: 처리량, 응답 시간, 오류율과 서버 한계를 측정한다.
- 브라우저 테스트: 렌더링, 사용자 동작과 페이지 JavaScript 실행을 확인한다.
- 분석 수집 테스트: 실제 사용자 동작으로 SDK 이벤트를 발생시키고 수집 시스템에서 도착을 확인한다.

브라우저 동작 성공만으로 PostHog 수집 성공을 판단하지 않는다. 테스트 이벤트를 조회해 이벤트 이름, 속성, 사용자 식별과 전환 연결까지 확인한다.

## 2. k6 하이브리드 구성

현재 목표가 API 부하와 PostHog 수집 검증을 함께 포함하므로 k6를 채택한다.

```text
다수의 k6/http 가상 사용자
→ 서버 API 부하 생성

소수의 k6/browser 가상 사용자
→ 실제 화면 동작
→ 프런트 SDK 이벤트 발생
→ PostHog 수집 결과 확인
```

브라우저 VU 수는 고정 비율로 결정하지 않는다. 부하 생성기 CPU·메모리와 시스템 특성을 측정하면서 최소 수부터 늘린다. API와 브라우저 결과는 scenario와 tag로 구분한다.

## 3. Gatling 재검토 조건

다음 조건으로 목표가 바뀌면 Gatling을 다시 비교한다.

- 실제 브라우저와 프런트 SDK 검증이 범위에서 제외된다.
- 서버 API의 성능과 한계만 측정한다.
- Java 코드와 Maven·Gradle 안에서 테스트를 관리하는 이점이 크다.
- 같은 조건의 시험에서 실행 비용이나 리포트가 현재 요구에 더 적합하다.

Gatling을 추가하면 브라우저 자동화 도구를 별도로 운영해야 한다. API 전용으로 전환되지 않는 한 도구 수 증가 비용이 남는다.

## 비용·제약

- 합성 이벤트는 운영 WAU와 전환율을 오염시킬 수 있다. 별도 PostHog 프로젝트나 테스트 표식을 사용하고 운영 지표에서 제외한다.
- 결제 흐름은 부하 도구가 해결하지 않는다. 결제사 Sandbox, 테스트 승인 경로 또는 대역을 별도로 설계한다.
- Chromium 검증은 모든 브라우저 호환성 검증을 대신하지 않는다.
- 부하 생성기 자체가 포화되면 서버 한계로 오판할 수 있다. 생성기 CPU·메모리·네트워크를 함께 관찰한다.
- 클라우드 실행 비용과 분산 실행 방식은 현재 **확인 필요**다.

## 검증 방법

- 같은 요청 비율, 데이터, 부하 모델, 연결 설정과 실행 환경으로 k6와 Gatling을 비교한다.
- HTTP VU만 실행한 기준값과 브라우저 VU를 함께 실행한 결과를 비교한다.
- 브라우저 수를 늘리며 부하 생성기 자원 포화 지점을 확인한다.
- 테스트 동작 뒤 PostHog에서 합성 이벤트와 전환 연결을 조회한다.
- 결제 Sandbox나 대역이 목표 처리량을 제한하지 않는지 확인한다.

## 기술 문서

- [k6 테스트 작성 언어](https://grafana.com/docs/k6/latest/get-started/write-your-first-test/)
- [k6 브라우저 API](https://grafana.com/docs/k6/latest/javascript-api/k6-browser/)
- [k6 하이브리드 성능 테스트](https://grafana.com/docs/k6/latest/using-k6-browser/recommended-practices/hybrid-approach-to-performance/)
- [Gatling HTTP 프로토콜](https://docs.gatling.io/reference/script/http/protocol/)
- [Gatling 지원 언어와 설치](https://docs.gatling.io/reference/deploy/install-local/)

## 최종 판단 기준

> 실제 브라우저와 프런트 분석 SDK를 API 부하와 함께 검증하면 k6의 하이브리드 구성을 사용한다. 서버 API만 검증하고 Java·Gradle 통합이 중요하면 Gatling을 재검토한다. 도구의 성능 우열은 동일한 부하 모델과 실행 환경에서 측정하며, 브라우저 VU는 소수로 제한한다.
