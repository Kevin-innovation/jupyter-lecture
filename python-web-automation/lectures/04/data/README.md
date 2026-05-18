# 레슨 04 데이터

이 폴더는 파일 다운로드와 폴더 정리 수업용 합성 fixture를 담고 있다. 실제 개인정보, 유료 자료, 외부 사이트 파일은 포함하지 않는다.

## 파일 구성

| 파일 | 용도 |
|---|---|
| resource_center.html | 자료실 링크 화면 예제. a.file-link에 href, data-type, data-week, data-group이 들어 있다 |
| manifest.csv | 다운로드 대상 파일명, 그룹, 타입, 주차, 경로를 담은 관리용 목록 |
| files/ | 저장 실습에 사용할 작은 샘플 파일 모음 |

## 수업 기준

- HTML 링크에서는 화면 텍스트와 href 경로를 구분한다.
- 실제 파일 저장은 bytes 기준으로 처리한다.
- 저장 파일명은 safe_filename으로 정리한다.
- 저장 후 파일 크기가 0보다 큰지 확인한다.
- download_log.csv와 download_summary.json처럼 재검토 가능한 산출물을 남긴다.
- 실제 사이트로 확장할 때는 권한, 최대 파일 수, 요청 간격, 저장 위치를 먼저 정한다.
