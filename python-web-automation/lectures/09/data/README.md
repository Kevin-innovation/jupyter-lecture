# 레슨 09 데이터

targets.csv는 수집 대상의 우선순위와 최소 요청 간격을 담는다. response_plan.csv는 endpoint별 시도 순서와 상태 코드를 담은 합성 응답 계획이다. robots_sample.txt는 Disallow 규칙 해석 연습용이다. payload_templates.json은 로그 요약 문구와 상태 설명을 제공한다.

응답은 모두 PlannedFetcher로 재현되는 합성 응답이다. 실제 외부 서버에 반복 요청하지 않는다.
## 파일별 사용 위치

- `targets.csv`: endpoint_id, url, priority, min_interval_seconds, owner를 포함한다. 학생은 priority 기준 정렬과 robots 차단 경로 필터링을 연습한다.
- `response_plan.csv`: 실제 서버 요청 대신 사용할 합성 응답 계획이다. 같은 endpoint를 여러 번 호출하면 순서대로 429, 503, 200 같은 상태가 재현된다.
- `robots_sample.txt`: `/private`, `/tmp` 차단 경로와 crawl-delay 예시가 들어 있다. 실제 robots.txt를 완전히 구현하는 수업이 아니라, 자동화 전에 접근 가능 범위를 확인하는 습관을 만드는 자료다.
- `payload_templates.json`: 최종 로그 요약 문장을 만들 때 사용한다.

## 검증 포인트

학생은 재시도 가능한 상태 코드와 즉시 중단해야 하는 상태 코드를 구분해야 한다. 무한 재시도, sleep 남발, 실패 숨김은 모두 보완 대상이다. 로그에는 endpoint, attempt_no, status_code, retry 여부가 남아야 한다.
