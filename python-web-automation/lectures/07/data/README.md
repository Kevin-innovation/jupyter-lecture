# 레슨 07 데이터

dynamic_dashboard.html은 data-delay-step으로 늦게 나타나는 요소를 흉내 낸 fixture다. selector_lab.html은 불안정한 class와 안정적인 data-testid를 비교하기 위한 파일이다. wait_cases.csv는 wait 검증 케이스다.

## 파일별 세부 설명

`dynamic_dashboard.html`은 `data-delay-step`을 사용해 요소가 늦게 보이는 상황을 흉내 낸다. metric-active는 즉시 보이고, metric-submit과 metric-pass는 step이 증가해야 보인다. student-row도 step에 따라 보이는 항목이 달라진다.

`selector_lab.html`은 일부러 불안정한 class와 안정적인 data-testid를 함께 제공한다. `.random-77` 같은 class는 예제로는 잡히지만 추천 selector가 아니다. 학생은 같은 요소를 여러 selector로 잡아 보고 어떤 기준이 유지보수에 좋은지 비교한다.

`wait_cases.csv`는 성공 케이스와 의도된 실패 케이스를 함께 담는다. expected_text가 빈 행은 selector가 끝까지 나타나지 않는 경우를 표현한다. 학생은 예외를 무조건 오류로만 보지 않고, 테스트 케이스의 기대 결과와 비교해 판단해야 한다.

모든 fixture는 수업용 합성 데이터다. 실제 학생 개인정보나 로그인 정보는 포함하지 않는다. 실제 사이트로 확장할 때는 테스트 계정, 요청 간격, 개인정보 로그 제한을 먼저 확인한다.
