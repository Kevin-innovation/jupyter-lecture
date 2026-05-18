# 레슨 03 데이터

이 폴더는 HTML 테이블과 리스트 데이터 정리 수업용 합성 fixture를 담고 있다. 실제 학생 개인정보나 외부 사이트 데이터는 포함하지 않는다.

## 파일 구성

| 파일 | 용도 |
|---|---|
| class_dashboard.html | 학생별 course, progress, submissions, passed, minutes, status를 담은 table 예제 |
| feedback_cards.html | article.feedback-card 구조와 data-priority 속성을 읽는 카드형 UI 예제 |
| todo_list.html | li.todo-item 구조와 data-status 속성을 읽는 리스트형 UI 예제 |
| rubric.csv | 최종 미션 평가 기준과 배점 예제 |

## 수업 기준

- table은 thead th를 먼저 읽고 tbody tr을 반복한다.
- card는 article.feedback-card 하나가 완성된 데이터 단위다.
- list는 li.todo-item을 반복하고 data-status를 함께 읽는다.
- progress, submissions, passed, minutes, max_score는 계산 전에 숫자로 변환한다.
- 실제 사이트로 확장할 때는 약관, robots.txt, 개인정보 여부, 요청 간격을 먼저 확인한다.
