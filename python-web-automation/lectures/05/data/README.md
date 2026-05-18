# 레슨 05 데이터

이 폴더는 세션과 쿠키 상태 관리 수업용 합성 fixture를 담고 있다. 실제 계정 정보, 실제 쿠키, 비밀번호, 외부 사이트 인증 정보는 포함하지 않는다.

## 파일 구성

| 파일 | 용도 |
|---|---|
| login_form.html | hidden csrf_token과 username/password input 구조를 읽는 폼 예제 |
| catalog_page.html | product-card, data-product-id, data-category, data-price를 읽는 카탈로그 예제 |
| cookie_policy.txt | 쿠키와 세션 실습 안전 규칙 |
| session_events.csv | login, filter, add, remove, checkout, logout 이벤트 흐름 예제 |

## 수업 기준

- 실제 사이트 로그인을 시도하지 않는다.
- 실제 쿠키나 비밀번호를 코드, 로그, 산출물에 저장하지 않는다.
- DemoSession의 합성 쿠키와 장바구니만 사용한다.
- 쿠키 값 원문 대신 cookie_keys처럼 구조만 요약한다.
- 이벤트 로그는 상태 변화 순서를 이해하기 위한 합성 데이터로만 사용한다.
