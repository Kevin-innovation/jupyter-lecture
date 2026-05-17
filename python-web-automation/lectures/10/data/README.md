# 레슨 10 데이터

portal_index.html은 합성 포털의 시작 페이지다. portal_notice.html, portal_courses.html, portal_downloads.html, portal_status.html은 각각 공지, 과정, 자료, 상태 정보를 담는다. download_manifest.csv와 quality_rules.json은 최종 리포트 검증에 사용한다.

포털은 수업용 합성 HTML이다. 외부 사이트 요청 없이 통합 자동화 흐름을 검증한다.
## 파일별 사용 위치

- `portal_index.html`: 통합 프로젝트의 시작 페이지다. `a[data-page]` 링크를 읽어 포털의 하위 페이지 목록을 만든다.
- `portal_notice.html`: 공지 카드 12개가 들어 있다. title, data-level, data-date를 추출한다.
- `portal_courses.html`: 과정명, 담당자, 학생 수, 상태가 table로 들어 있다. 학생 수는 `clean_int()`로 숫자화한다.
- `portal_downloads.html`와 `download_manifest.csv`: 화면 카드와 CSV manifest를 비교하면서 자료 개수를 정리한다.
- `portal_status.html`: 실행, 제출, 오류 metric을 제공한다.
- `quality_rules.json`: 리포트 대상 과정 필터링 기준을 담는다.

## 검증 포인트

최종 레슨은 여러 입력을 하나의 운영 리포트로 묶는 것이 핵심이다. 학생은 링크 수집, 표 파싱, CSV 읽기, JSON 규칙 적용, CSV/JSON/SQLite 저장, 3문장 운영 메모까지 한 흐름으로 설명할 수 있어야 한다.
