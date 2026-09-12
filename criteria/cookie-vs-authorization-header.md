# Cookie와 Authorization 헤더의 인증 정보 전달 선택

- 상태: draft
- 주제: HTTP 요청의 인증 정보 전달 위치
- 판단 축: 클라이언트 종류, 자동 전송 여부, JavaScript 접근, CSRF, CORS

## 결론

**채택: 동일 출처 브라우저의 세션 ID는 Cookie로 전달한다.** Cookie에는 HttpOnly, Secure, SameSite 속성을 설정하고 상태 변경 요청에는 CSRF 방어를 적용한다.

**채택: 모바일 앱과 외부 API의 access token은 Authorization: Bearer 헤더로 전달한다.** 클라이언트가 요청마다 명시적으로 붙인다. Cookie와 헤더를 함께 지원하면 요청별 인증 주체 선택 규칙을 명시한다.

**거부: access token을 URL query, 로그, 오류 응답에 넣는다.** URL은 브라우저 기록·프록시·접근 로그에 남을 수 있다.

## 빠른 기준

| 질문 | 판단 |
| --- | --- |
| 동일 출처 브라우저가 서버 세션을 사용한다 | Cookie |
| 모바일 앱·외부 API가 access token을 명시적으로 전달한다 | Authorization: Bearer |
| JWT를 Cookie에 넣고 싶다 | 가능하지만 Cookie의 자동 전송·CSRF 규칙을 함께 적용 |
| 브라우저 JavaScript가 token을 직접 보관·첨부한다 | XSS 탈취 위험과 짧은 수명·저장 위치를 함께 설계 |
| 한 요청에 Cookie와 헤더 credential이 함께 있다 | 명시한 우선순위로 하나만 사용하거나 거부. 추측으로 fallback하지 않음 |
| access token을 query parameter로 전달한다 | 거부 |

## 1. 적용 범위

이 기준은 인증 정보를 HTTP 요청 어디에 넣을지를 다룬다. 세션과 토큰 중 무엇으로 인증 상태를 검증할지는 [세션과 토큰 기반 인증 상태 선택](session-vs-token.md)에서 다룬다.

~~~text
인증 상태 검증: 세션인가, 토큰인가
전달 위치: Cookie인가, Authorization 헤더인가
~~~

세션 ID는 Cookie로, access token은 Authorization 헤더로 전달하는 조합이 일반적이다. 하지만 JWT를 Cookie에 넣거나 세션 ID를 별도 헤더에 넣는 것도 기술적으로 가능하다. 이때 보안 속성은 전달 위치를 따른다.

## 2. 두 방식의 차이

| 항목 | Cookie | Authorization 헤더 |
| --- | --- | --- |
| 요청 첨부 | 브라우저가 범위에 맞으면 자동 전송 | 클라이언트가 요청마다 명시적으로 첨부 |
| 주 사용처 | 동일 출처 브라우저 세션 | 모바일 앱, 외부 API, 명시적 access token |
| CSRF | 자동 전송 때문에 방어 필요 | 브라우저 교차 출처 요청에는 CORS 정책을 함께 검토 |
| XSS | HttpOnly면 JavaScript 읽기 차단. 단, 사용자 권한 요청 실행은 가능 | 브라우저 JavaScript가 읽을 수 있는 저장소는 탈취 위험 |
| 교차 출처 | Cookie 범위·SameSite·credentials 설정 검토 | CORS preflight와 허용 헤더 검토 |
| 서버 해석 | Cookie 이름과 속성 계약 | Authorization scheme과 Bearer 형식 계약 |

Cookie와 헤더는 보안 우열이 아니라 위협 모델과 클라이언트 동작의 차이다. HttpOnly Cookie도 XSS가 있으면 공격자가 로그인 사용자의 요청을 실행할 수 있다. Authorization 헤더도 브라우저 JavaScript가 token을 읽을 수 있으면 XSS로 탈취될 수 있다.

## 3. Cookie 선택

Cookie는 브라우저 세션의 식별자를 전달할 때 선택한다. 서버는 Cookie 값 자체가 아니라 해당 값으로 찾은 서버 세션을 인증 상태의 원본으로 사용한다.

다음 조건이면 선택한다.

- 브라우저가 주 클라이언트다.
- 인증 정보를 JavaScript에서 읽지 못하게 해야 한다.
- 동일 출처 요청에서 브라우저의 자동 전송을 활용한다.
- CSRF 방어와 Cookie 속성을 운영할 수 있다.

비브라우저 클라이언트가 주 사용처이거나 요청마다 인증 정보를 명시적으로 제어해야 한다면 Cookie를 기본 선택으로 두지 않는다.

~~~text
Set-Cookie: sessionId=...; HttpOnly; Secure; SameSite=Lax
~~~

필수 판단:

- HttpOnly: JavaScript의 Cookie 읽기를 막는다.
- Secure: HTTPS 요청에서만 전송한다.
- SameSite: 교차 사이트 자동 전송을 줄이는 방어층이다.
- CSRF 방어: 상태 변경 요청에 CSRF token, Origin·Referer 검증 또는 선택한 프레임워크의 방어를 적용한다.

**거부: SameSite만으로 CSRF 방어가 끝난다고 판단한다.** SameSite는 배포 형태와 HTTP 메서드에 따라 우회 여지가 있다. 특히 상태를 변경하는 GET API는 만들지 않는다.

JWT를 Cookie에 넣는 경우에도 Cookie 자동 전송과 CSRF 위험은 그대로 적용된다. 토큰 형식이 JWT라는 이유로 Header 방식의 보안 성질을 얻지 않는다.

## 4. Authorization 헤더 선택

모바일 앱과 외부 API는 access token을 다음 형식으로 보낸다.

~~~http
Authorization: Bearer {access-token}
~~~

다음 조건이면 선택한다.

- 모바일 앱, 외부 API, 서버 간 호출이 주 사용처다.
- 클라이언트가 요청마다 인증 정보를 명시적으로 첨부할 수 있다.
- Cookie의 자동 전송과 무관한 API 인증 계약이 필요하다.
- token 저장 위치, 수명, 재발급, 로그 마스킹을 운영할 수 있다.

브라우저에서 token을 영속 저장하기 위해 JavaScript 접근 가능한 저장소를 반드시 사용해야 하고 XSS 탈취 대응이 부족하다면 기본 선택으로 두지 않는다.

헤더 방식을 선택하면 클라이언트의 token 저장 위치, token 수명, 재발급, 로그 제외 정책을 함께 검증한다.

Bearer token은 가진 사람이 사용할 수 있는 자격 증명이다. HTTPS를 사용하고 프록시·애플리케이션 로그에서 Authorization 값을 마스킹한다.

## 5. 실패 응답과 인가의 구분

| 상황 | 응답 |
| --- | --- |
| Cookie·헤더가 없거나 credential 형식이 틀림 | 401 |
| 토큰 서명 오류·만료·무효화 | 401 |
| 인증은 됐지만 현재 자원 접근 권한이 없음 | 403 또는 리소스 은닉 정책에 따른 응답 |

전달 위치가 달라도 인증 실패 응답은 통일한다. 자원 인가 실패를 인증 실패로 뭉개지 않는다.

## 6. 검증 방법

- Cookie 기반 요청에서 HttpOnly, Secure, SameSite가 의도한 값으로 내려가는지 확인한다.
- Cookie 기반 상태 변경 요청에 CSRF 방어가 적용되는지 확인한다.
- Authorization 헤더의 누락, Bearer 접두어 누락, 빈 token, 만료·위조 token이 401인지 확인한다.
- access token이 URL, 예외 메시지, 애플리케이션·프록시 로그에 남지 않는지 확인한다.
- 교차 출처를 지원하면 Cookie credentials, 허용 Origin, 허용 헤더를 실제 브라우저 요청으로 확인한다.
- 인증 성공 뒤 자원 인가 실패가 403 또는 정한 은닉 응답인지 확인한다.

## 최종 판단 기준

> Cookie와 Authorization 헤더는 인증 상태를 전달하는 서로 다른 경로다. 동일 출처 브라우저 세션은 보안 속성과 CSRF 방어를 갖춘 Cookie로, 모바일·외부 API access token은 Authorization: Bearer 헤더로 전달한다. JWT를 Cookie에 넣으면 Cookie의 위험을, 헤더 token을 브라우저에 보관하면 XSS 탈취 위험을 함께 부담한다.

## 검증 범위

- 확인 필요: 적용 서비스의 Cookie 속성, CSRF 방어, token 저장 위치, CORS 설정과 로그 마스킹
- 비범위: 세션·토큰의 상태 관리 방식, 자원별 인가 규칙
