# 레슨 02 데이터

이 폴더는 URL 파라미터와 페이지네이션 수업에서 사용하는 합성 fixture를 담는다. 실제 사이트의 개인정보, 로그인 정보, 유료 콘텐츠는 포함하지 않는다.

## 파일 목록

- search_page_1.html: 자료실 검색 결과 1페이지. article.result-card 9개를 포함한다.
- search_page_2.html: 자료실 검색 결과 2페이지. 같은 구조의 카드 9개를 포함한다.
- search_page_3.html: 자료실 검색 결과 3페이지. 같은 구조의 카드 9개를 포함한다.
- search_targets.csv: 검색어, 카테고리, 최소 조회수, 최대 페이지 수를 담은 검색 계획 파일이다.
- robots_sample.txt: robots.txt와 요청 간격 개념을 설명하기 위한 수업용 샘플이다.

## selector 기준

검색 결과의 반복 단위는 article.result-card다. 제목과 대표 링크는 .title a, 상세 링크는 a.detail, 날짜는 time의 datetime 속성, 조회수는 .views에서 읽는다. 카테고리, 페이지 번호, 순위는 각각 data-category, data-page, data-rank 속성에 들어 있다.

## 수업 운영 메모

학생이 실제 사이트를 요청하지 않고도 페이지네이션과 query string 흐름을 연습하도록 만든 데이터다. 수업 중에는 파일을 반복해서 읽고, 실제 사이트로 확장할 때는 약관, robots 정책, 요청 간격을 먼저 확인한다.
