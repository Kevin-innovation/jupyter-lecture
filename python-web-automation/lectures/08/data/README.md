# 레슨 08 데이터

raw_product_feed.csv는 의도적으로 누락, 중복, 형식 오류가 섞인 합성 상품 피드다. category_rules.json은 허용 카테고리, 상태, 최대 가격 기준을 담는다. validation_panel.html은 같은 데이터를 웹 표 형태로 확인하는 연습용 페이지다. schema_notes.txt는 사람이 읽는 스키마 설명이다.

모든 데이터는 수업용 합성 데이터이며 실제 상품이나 개인정보를 포함하지 않는다. 오류 행은 검증 연습을 위해 의도적으로 포함했다.
## 파일별 사용 위치

- `raw_product_feed.csv`: 1~13번 문제에서 원본 행 수, 필수 컬럼, 가격/재고 정규화, 중복 record_id 탐지에 사용한다. 일부 행에는 빈 제목, 잘못된 가격 문자열, 허용되지 않은 category, 음수 stock, 허용되지 않은 status가 포함되어 있다.
- `category_rules.json`: 검증 함수가 읽는 기준 파일이다. required_fields, valid_categories, valid_status, max_price를 학생이 직접 확인하도록 한다.
- `validation_panel.html`: HTML table 구조에서 `tbody tr`를 선택하는 연습에 사용한다. CSV와 같은 데이터를 웹 화면 형태로 볼 때 selector를 어떻게 잡는지 설명한다.
- `schema_notes.txt`: 사람이 읽는 스키마 문서다. 자동화 코드와 운영 문서가 같은 기준을 공유해야 한다는 점을 설명할 때 사용한다.

## 검증 포인트

학생은 저장 전 검증을 반드시 거쳐야 한다. 원본 행 수와 정제 행 수가 다른 이유를 말할 수 있어야 하며, 중복과 오류 행을 조용히 버리지 않고 리포트에 남겨야 한다.
