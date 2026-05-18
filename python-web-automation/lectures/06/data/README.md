# 레슨 06 데이터

form_page.html은 입력과 버튼 클릭을 연습하는 합성 폼이다. todo_app.html은 클릭 후 상태 변화 확인용 fixture다. tabs_page.html은 탭 버튼과 패널 전환을 연습한다. form_cases.csv는 여러 입력 케이스 자동화를 위한 데이터다.

## 파일별 세부 설명

`form_page.html`에는 id와 data-testid가 모두 들어 있다. id는 수업 초반에 짧게 접근하기 좋고, data-testid는 실제 테스트 자동화에서 유지보수하기 좋은 선택자다. `#ready-message`는 data-delay-step 속성을 가지고 있어 wait 연습에 사용할 수 있다.

`todo_app.html`은 pending과 done 상태가 섞인 8개 항목을 제공한다. 버튼은 data-target으로 변경할 li를 가리킨다. 클릭 전후 같은 target의 data-status를 비교하면 상태 전환이 실제로 일어났는지 확인할 수 있다.

`tabs_page.html`은 Python, Web, Report 세 개의 panel을 제공한다. tab 버튼의 data-target 값과 panel id가 연결되어 있다. 실제 사이트에서는 aria-selected나 CSS display를 쓸 수 있지만, 이 fixture는 탭 전환의 핵심 구조를 단순화한다.

`form_cases.csv`는 10개 케이스를 담는다. 한 번 성공하는 자동화보다 여러 입력을 안정적으로 반복 처리하는 연습이 중요하므로, 학생은 cases 길이와 outputs 길이가 같은지 확인해야 한다.
