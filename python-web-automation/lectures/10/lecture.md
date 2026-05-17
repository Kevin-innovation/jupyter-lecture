---
jupyter:
  jupytext:
    text_representation:
      extension: .md
      format_name: markdown
      format_version: '1.3'
  kernelspec:
    display_name: Python 3
    language: python
    name: python3
---

# 레슨 10 — 통합 프로젝트: 자동화 리포트 만들기

[![Open in Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/Kevin-innovation/jupyter-lecture/blob/main/python-web-automation/lectures/10/[학생용] 레슨 10 — 통합 프로젝트: 자동화 리포트 만들기.ipynb)

이 노트북은 읽기와 따라하기용 강의 노트북이다. HTML 파싱, 상대 URL, 자료 목록, 품질 검증, 저장, 로그 요약을 하나의 작은 업무 자동화 리포트로 묶는 최종 프로젝트을 안전한 합성 fixture로 연습한다.

## 학습 목표

1. 여러 HTML fixture를 하나의 포털 구조로 해석한다.
2. 링크, 공지, 과정, 다운로드 자료를 각각 추출한다.
3. 중복과 상태 오류를 점검해 저장 전 품질을 확인한다.
4. CSV, JSON, SQLite 산출물을 함께 만든다.
5. 운영자가 읽을 수 있는 3문장 자동화 메모를 작성한다.

---

## 1. 수업 맥락과 안전 기준

마지막 레슨은 “한 페이지에서 값을 뽑았다”가 아니라 “업무 담당자가 바로 확인할 수 있는 리포트”까지 만든다. 합성 포털을 대상으로 하므로 외부 사이트 부하 없이 실제 운영 흐름을 통합 연습한다.

자동화는 빠르게 반복하는 도구이기 때문에 실패했을 때 더 위험해질 수 있다. 그래서 이번 레슨에서는 모든 입력을 수업용 파일로 고정하고, 결과를 저장하기 전에 검증하거나 로그를 남기는 과정을 코드에 포함한다. 이 습관은 실제 사이트를 대상으로 할 때 요청량을 줄이고, 오류를 빨리 발견하게 만든다.

## 2. 환경 셀

~~~python
import os
import re
import csv
import json
import time
import sqlite3
import logging
from pathlib import Path
from urllib.parse import urljoin, urlparse

try:
    import requests
    from bs4 import BeautifulSoup
except ImportError:
    import sys, subprocess
    subprocess.check_call([sys.executable, '-m', 'pip', '-q', 'install', 'requests', 'beautifulsoup4'])
    import requests
    from bs4 import BeautifulSoup

IS_COLAB = 'COLAB_GPU' in os.environ or 'COLAB_TPU_ADDR' in os.environ
if IS_COLAB:
    DATA_BASE = 'https://raw.githubusercontent.com/Kevin-innovation/jupyter-lecture/main/python-web-automation/lectures/10/data'
else:
    DATA_BASE = './data'

def load_text(filename):
    if DATA_BASE.startswith('http'):
        url = f'{DATA_BASE}/{filename}'
        response = requests.get(url, timeout=10, headers={'User-Agent': 'D-Lab-Lesson/1.0'})
        response.raise_for_status()
        response.encoding = response.encoding or 'utf-8'
        return response.text
    return Path(DATA_BASE, filename).read_text(encoding='utf-8')

def load_csv(filename):
    text = load_text(filename)
    return list(csv.DictReader(text.splitlines()))

def load_json(filename):
    return json.loads(load_text(filename))

def clean_int(text):
    return int(re.sub(r'[^0-9]', '', str(text)) or '0')

def write_csv(path, rows, fieldnames=None):
    rows = list(rows)
    if fieldnames is None:
        fieldnames = list(rows[0].keys()) if rows else []
    with open(path, 'w', encoding='utf-8', newline='') as f:
        writer = csv.DictWriter(f, fieldnames=fieldnames)
        writer.writeheader()
        writer.writerows(rows)
    return path

print('colab:', IS_COLAB)
print('data base:', DATA_BASE)

def parse_html(filename):
    return BeautifulSoup(load_text(filename), 'html.parser')

def text_or_empty(el):
    return '' if el is None else el.get_text(' ', strip=True)

def status_ok(status):
    return str(status).strip().lower() in {'open', 'ready', 'published'}

def make_key(*parts):
    return '::'.join(str(part).strip().lower() for part in parts)
~~~

---

## 3. 핵심 개념

이 셀은 포털 시작 페이지의 제목을 읽으며 프로젝트 입력을 확인한다. 학생은 시작점이 무엇인지 명확히 잡은 뒤 하위 페이지로 이동한다.

~~~python
index = parse_html('portal_index.html')
print(text_or_empty(index.select_one('h1')))
~~~
시작 페이지 확인은 통합 프로젝트의 기준점을 만든다.

---

## 4. 자료 구조 확인

목차 링크 수집은 사이트맵을 만드는 첫 단계다. label과 href를 함께 저장하면 다음 처리 순서를 사람이 읽을 수 있다.

~~~python
links = []
for a in index.select('a[data-page]'):
    links.append({'label': a.get_text(' ', strip=True), 'href': a['href'], 'url': urljoin('https://lesson.local/portal/', a['href'])})
print(links)
~~~
링크 목록은 이후 처리할 작업 큐 역할을 한다.

---

## 5. 품질 기준 적용

상대 URL을 절대 URL로 바꾸는 과정은 링크 자동화의 기본이다. 이 원칙은 다운로드 자료와 상세 페이지 수집에도 그대로 이어진다.

~~~python
notice = parse_html('portal_notice.html')
notices = [{'title': item.select_one('.title').get_text(' ', strip=True), 'level': item.get('data-level'), 'date': item.get('data-date')} for item in notice.select('.notice-card')]
print(notices[:2])
~~~
절대 URL 변환은 다른 페이지로 이동할 때 경로 오류를 줄인다.

---

## 6. 저장과 보고

공지 카드는 반복 단위가 명확한 HTML 구조다. 학생은 카드 개수를 먼저 확인한 뒤 제목, 레벨, 날짜를 분리한다.

~~~python
courses = parse_html('portal_courses.html')
course_rows = []
for row in courses.select('tbody tr'):
    cells = [td.get_text(' ', strip=True) for td in row.select('td')]
    course_rows.append({'course': cells[0], 'teacher': cells[1], 'students': clean_int(cells[2]), 'status': cells[3]})
print(course_rows[:2])
~~~
카드 개수 확인은 selector가 맞는지 검증하는 빠른 방법이다.

---

## 7. 운영 관점 점검

과정 표는 셀 순서를 읽어 딕셔너리로 바꾸는 연습이다. 학생 수 문자열을 숫자로 정리해야 이후 필터링이 가능하다.

~~~python
manifest = load_csv('download_manifest.csv')
print(manifest[:2])
~~~
표 행 정리는 과정별 요약을 만드는 기반이다.

---

## 8. 마무리 체크

manifest CSV는 화면 자료와 저장 기준을 연결한다. 파일명과 course를 key로 삼으면 과정별 자료 개수를 안정적으로 계산할 수 있다.

~~~python
rules = load_json('quality_rules.json')
ready_courses = [row for row in course_rows if status_ok(row['status']) and row['students'] >= rules['min_students_for_report']]
print(ready_courses)
~~~
manifest는 자료 누락 여부를 확인하는 기준 파일이다.

---

## 9. 핵심 개념

품질 규칙 JSON은 리포트 대상 과정을 고르는 기준이다. 코드 안에 숫자를 직접 박지 않고 설정 파일로 분리하는 이유를 설명한다.

~~~python
report = {'notice_count': len(notices), 'course_count': len(course_rows), 'file_count': len(manifest), 'ready_course_count': len(ready_courses)}
Path('lesson10_portal_report.json').write_text(json.dumps(report, ensure_ascii=False, indent=2), encoding='utf-8')
print(report)
~~~
설정 JSON은 리포트 조건을 코드 밖으로 분리한다.

---

## 데이터 출처와 안전 규칙

portal_index.html은 합성 포털의 시작 페이지다. portal_notice.html, portal_courses.html, portal_downloads.html, portal_status.html은 각각 공지, 과정, 자료, 상태 정보를 담는다. download_manifest.csv와 quality_rules.json은 최종 리포트 검증에 사용한다.

- 모든 파일은 수업용 합성 데이터다.
- 실제 사이트에 반복 요청하지 않는다.
- 저장 파일은 레슨 폴더 또는 코랩 현재 작업 폴더에만 만든다.
- 외부 사이트로 확장할 때는 약관, robots.txt, 요청 간격, 개인정보 포함 여부를 먼저 확인한다.

---

## 강의 보강 노트

이 절은 수업 중 교사가 질문으로 풀어낼 수 있는 운영형 설명이다. 학생이 셀을 실행한 뒤 결과만 맞히지 않고 자동화 절차를 말로 설명하도록 돕는다.

### 1. 통합 프로젝트의 기준

마지막 레슨은 개별 기술을 모두 섞는 것이 아니라 입력, 처리, 검증, 저장, 요약의 흐름을 하나의 리포트로 완성하는 단계다. 학생에게 코드를 길게 쓰기보다 단계 이름을 분명히 붙이게 한다.

### 2. 포털 링크 수집

index 페이지의 링크를 읽는 과정은 사이트맵을 만드는 연습이다. 상대 URL을 절대 URL로 바꾸고, label과 href를 함께 저장하면 이후 페이지별 처리 계획을 세울 수 있다.

### 3. 공지와 과정의 결합

공지 카드와 과정 표는 형태가 다르지만 최종 리포트에서는 같은 운영 상황을 설명한다. 서로 다른 입력을 하나의 summary로 묶는 연습이 이 프로젝트의 핵심이다.

### 4. 자료 manifest 검증

다운로드 화면과 manifest CSV를 비교하면 화면에 보이는 자료와 저장 기준이 어긋나는지 확인할 수 있다. 실제 운영에서는 이 단계가 누락 자료를 찾는 데 중요하다.

### 5. 상태 metric 활용

실행 수, 제출 수, 오류 수 같은 metric은 단순 장식이 아니라 리포트의 신뢰도를 설명하는 근거다. 학생에게 숫자 하나 이상을 운영 메모에 포함하게 한다.

### 6. 최종 산출물

CSV는 상세 목록, JSON은 요약, SQLite는 조회 가능한 저장소 역할을 한다. 세 파일을 모두 만들 필요는 없지만 각각의 목적을 구분할 수 있어야 한다.

### 7. 발표 기준

프로젝트 제출 후 학생은 어떤 파일을 읽었고, 어떤 기준으로 필터링했으며, 어떤 산출물을 만들었는지 1분 안에 설명해야 한다. 설명이 가능하면 코드 구조도 대체로 안정적이다.

### 체크포인트 1: 입력 파일 검증한다

수업 중 피드백 시간을 줄이기 위해, 레슨 10 통합 프로젝트: 자동화 리포트 만들기에서는 입력 파일을/를 검증한다는 과정을 별도의 단계로 둔다. 이때 학생은 화면에 보이는 값 하나보다 어떤 입력에서 어떤 변환을 거쳐 결과가 나왔는지 설명해야 한다. 강사는 변수명, 출력 형태, 저장 여부를 함께 확인하며 다음 실행에서도 같은 결과가 나오는지 묻는다.

### 체크포인트 2: 선택자 확인한다

학생이 막힌 지점을 찾기 위해, 레슨 10 통합 프로젝트: 자동화 리포트 만들기에서는 선택자을/를 확인한다는 과정을 별도의 단계로 둔다. 이때 학생은 화면에 보이는 값 하나보다 어떤 입력에서 어떤 변환을 거쳐 결과가 나왔는지 설명해야 한다. 강사는 변수명, 출력 형태, 저장 여부를 함께 확인하며 다음 실행에서도 같은 결과가 나오는지 묻는다.

### 체크포인트 3: 오류 메시지 분리한다

운영자가 결과를 이해할 수 있게, 레슨 10 통합 프로젝트: 자동화 리포트 만들기에서는 오류 메시지을/를 분리한다는 과정을 별도의 단계로 둔다. 이때 학생은 화면에 보이는 값 하나보다 어떤 입력에서 어떤 변환을 거쳐 결과가 나왔는지 설명해야 한다. 강사는 변수명, 출력 형태, 저장 여부를 함께 확인하며 다음 실행에서도 같은 결과가 나오는지 묻는다.

### 체크포인트 4: 결과 행 수 기록한다

재실행했을 때 같은 결과를 얻기 위해, 레슨 10 통합 프로젝트: 자동화 리포트 만들기에서는 결과 행 수을/를 기록한다는 과정을 별도의 단계로 둔다. 이때 학생은 화면에 보이는 값 하나보다 어떤 입력에서 어떤 변환을 거쳐 결과가 나왔는지 설명해야 한다. 강사는 변수명, 출력 형태, 저장 여부를 함께 확인하며 다음 실행에서도 같은 결과가 나오는지 묻는다.

### 체크포인트 5: 검증 기준 비교한다

다음 셀에서 재사용하기 위해, 레슨 10 통합 프로젝트: 자동화 리포트 만들기에서는 검증 기준을/를 비교한다는 과정을 별도의 단계로 둔다. 이때 학생은 화면에 보이는 값 하나보다 어떤 입력에서 어떤 변환을 거쳐 결과가 나왔는지 설명해야 한다. 강사는 변수명, 출력 형태, 저장 여부를 함께 확인하며 다음 실행에서도 같은 결과가 나오는지 묻는다.

### 체크포인트 6: 중간 변수 요약한다

실제 사이트 확장 전에 위험을 낮추기 위해, 레슨 10 통합 프로젝트: 자동화 리포트 만들기에서는 중간 변수을/를 요약한다는 과정을 별도의 단계로 둔다. 이때 학생은 화면에 보이는 값 하나보다 어떤 입력에서 어떤 변환을 거쳐 결과가 나왔는지 설명해야 한다. 강사는 변수명, 출력 형태, 저장 여부를 함께 확인하며 다음 실행에서도 같은 결과가 나오는지 묻는다.

### 체크포인트 7: 입력 파일 검토한다

저장 파일의 신뢰도를 높이기 위해, 레슨 10 통합 프로젝트: 자동화 리포트 만들기에서는 입력 파일을/를 검토한다는 과정을 별도의 단계로 둔다. 이때 학생은 화면에 보이는 값 하나보다 어떤 입력에서 어떤 변환을 거쳐 결과가 나왔는지 설명해야 한다. 강사는 변수명, 출력 형태, 저장 여부를 함께 확인하며 다음 실행에서도 같은 결과가 나오는지 묻는다.

### 체크포인트 8: 선택자 정리한다

코랩과 로컬 실행 차이를 줄이기 위해, 레슨 10 통합 프로젝트: 자동화 리포트 만들기에서는 선택자을/를 정리한다는 과정을 별도의 단계로 둔다. 이때 학생은 화면에 보이는 값 하나보다 어떤 입력에서 어떤 변환을 거쳐 결과가 나왔는지 설명해야 한다. 강사는 변수명, 출력 형태, 저장 여부를 함께 확인하며 다음 실행에서도 같은 결과가 나오는지 묻는다.

### 체크포인트 9: 오류 메시지 설명한다

수업 중 피드백 시간을 줄이기 위해, 레슨 10 통합 프로젝트: 자동화 리포트 만들기에서는 오류 메시지을/를 설명한다는 과정을 별도의 단계로 둔다. 이때 학생은 화면에 보이는 값 하나보다 어떤 입력에서 어떤 변환을 거쳐 결과가 나왔는지 설명해야 한다. 강사는 변수명, 출력 형태, 저장 여부를 함께 확인하며 다음 실행에서도 같은 결과가 나오는지 묻는다.

### 체크포인트 10: 결과 행 수 저장한다

학생이 막힌 지점을 찾기 위해, 레슨 10 통합 프로젝트: 자동화 리포트 만들기에서는 결과 행 수을/를 저장한다는 과정을 별도의 단계로 둔다. 이때 학생은 화면에 보이는 값 하나보다 어떤 입력에서 어떤 변환을 거쳐 결과가 나왔는지 설명해야 한다. 강사는 변수명, 출력 형태, 저장 여부를 함께 확인하며 다음 실행에서도 같은 결과가 나오는지 묻는다.

### 체크포인트 11: 검증 기준 되돌아본다

운영자가 결과를 이해할 수 있게, 레슨 10 통합 프로젝트: 자동화 리포트 만들기에서는 검증 기준을/를 되돌아본다는 과정을 별도의 단계로 둔다. 이때 학생은 화면에 보이는 값 하나보다 어떤 입력에서 어떤 변환을 거쳐 결과가 나왔는지 설명해야 한다. 강사는 변수명, 출력 형태, 저장 여부를 함께 확인하며 다음 실행에서도 같은 결과가 나오는지 묻는다.

### 체크포인트 12: 중간 변수 표준화한다

재실행했을 때 같은 결과를 얻기 위해, 레슨 10 통합 프로젝트: 자동화 리포트 만들기에서는 중간 변수을/를 표준화한다는 과정을 별도의 단계로 둔다. 이때 학생은 화면에 보이는 값 하나보다 어떤 입력에서 어떤 변환을 거쳐 결과가 나왔는지 설명해야 한다. 강사는 변수명, 출력 형태, 저장 여부를 함께 확인하며 다음 실행에서도 같은 결과가 나오는지 묻는다.

### 체크포인트 13: 입력 파일 검증한다

다음 셀에서 재사용하기 위해, 레슨 10 통합 프로젝트: 자동화 리포트 만들기에서는 입력 파일을/를 검증한다는 과정을 별도의 단계로 둔다. 이때 학생은 화면에 보이는 값 하나보다 어떤 입력에서 어떤 변환을 거쳐 결과가 나왔는지 설명해야 한다. 강사는 변수명, 출력 형태, 저장 여부를 함께 확인하며 다음 실행에서도 같은 결과가 나오는지 묻는다.

### 체크포인트 14: 선택자 확인한다

실제 사이트 확장 전에 위험을 낮추기 위해, 레슨 10 통합 프로젝트: 자동화 리포트 만들기에서는 선택자을/를 확인한다는 과정을 별도의 단계로 둔다. 이때 학생은 화면에 보이는 값 하나보다 어떤 입력에서 어떤 변환을 거쳐 결과가 나왔는지 설명해야 한다. 강사는 변수명, 출력 형태, 저장 여부를 함께 확인하며 다음 실행에서도 같은 결과가 나오는지 묻는다.

### 체크포인트 15: 오류 메시지 분리한다

저장 파일의 신뢰도를 높이기 위해, 레슨 10 통합 프로젝트: 자동화 리포트 만들기에서는 오류 메시지을/를 분리한다는 과정을 별도의 단계로 둔다. 이때 학생은 화면에 보이는 값 하나보다 어떤 입력에서 어떤 변환을 거쳐 결과가 나왔는지 설명해야 한다. 강사는 변수명, 출력 형태, 저장 여부를 함께 확인하며 다음 실행에서도 같은 결과가 나오는지 묻는다.

### 체크포인트 16: 결과 행 수 기록한다

코랩과 로컬 실행 차이를 줄이기 위해, 레슨 10 통합 프로젝트: 자동화 리포트 만들기에서는 결과 행 수을/를 기록한다는 과정을 별도의 단계로 둔다. 이때 학생은 화면에 보이는 값 하나보다 어떤 입력에서 어떤 변환을 거쳐 결과가 나왔는지 설명해야 한다. 강사는 변수명, 출력 형태, 저장 여부를 함께 확인하며 다음 실행에서도 같은 결과가 나오는지 묻는다.

### 체크포인트 17: 검증 기준 비교한다

수업 중 피드백 시간을 줄이기 위해, 레슨 10 통합 프로젝트: 자동화 리포트 만들기에서는 검증 기준을/를 비교한다는 과정을 별도의 단계로 둔다. 이때 학생은 화면에 보이는 값 하나보다 어떤 입력에서 어떤 변환을 거쳐 결과가 나왔는지 설명해야 한다. 강사는 변수명, 출력 형태, 저장 여부를 함께 확인하며 다음 실행에서도 같은 결과가 나오는지 묻는다.

### 체크포인트 18: 중간 변수 요약한다

학생이 막힌 지점을 찾기 위해, 레슨 10 통합 프로젝트: 자동화 리포트 만들기에서는 중간 변수을/를 요약한다는 과정을 별도의 단계로 둔다. 이때 학생은 화면에 보이는 값 하나보다 어떤 입력에서 어떤 변환을 거쳐 결과가 나왔는지 설명해야 한다. 강사는 변수명, 출력 형태, 저장 여부를 함께 확인하며 다음 실행에서도 같은 결과가 나오는지 묻는다.

### 체크포인트 19: 입력 파일 검토한다

운영자가 결과를 이해할 수 있게, 레슨 10 통합 프로젝트: 자동화 리포트 만들기에서는 입력 파일을/를 검토한다는 과정을 별도의 단계로 둔다. 이때 학생은 화면에 보이는 값 하나보다 어떤 입력에서 어떤 변환을 거쳐 결과가 나왔는지 설명해야 한다. 강사는 변수명, 출력 형태, 저장 여부를 함께 확인하며 다음 실행에서도 같은 결과가 나오는지 묻는다.

### 체크포인트 20: 선택자 정리한다

재실행했을 때 같은 결과를 얻기 위해, 레슨 10 통합 프로젝트: 자동화 리포트 만들기에서는 선택자을/를 정리한다는 과정을 별도의 단계로 둔다. 이때 학생은 화면에 보이는 값 하나보다 어떤 입력에서 어떤 변환을 거쳐 결과가 나왔는지 설명해야 한다. 강사는 변수명, 출력 형태, 저장 여부를 함께 확인하며 다음 실행에서도 같은 결과가 나오는지 묻는다.

### 체크포인트 21: 오류 메시지 설명한다

다음 셀에서 재사용하기 위해, 레슨 10 통합 프로젝트: 자동화 리포트 만들기에서는 오류 메시지을/를 설명한다는 과정을 별도의 단계로 둔다. 이때 학생은 화면에 보이는 값 하나보다 어떤 입력에서 어떤 변환을 거쳐 결과가 나왔는지 설명해야 한다. 강사는 변수명, 출력 형태, 저장 여부를 함께 확인하며 다음 실행에서도 같은 결과가 나오는지 묻는다.

### 체크포인트 22: 결과 행 수 저장한다

실제 사이트 확장 전에 위험을 낮추기 위해, 레슨 10 통합 프로젝트: 자동화 리포트 만들기에서는 결과 행 수을/를 저장한다는 과정을 별도의 단계로 둔다. 이때 학생은 화면에 보이는 값 하나보다 어떤 입력에서 어떤 변환을 거쳐 결과가 나왔는지 설명해야 한다. 강사는 변수명, 출력 형태, 저장 여부를 함께 확인하며 다음 실행에서도 같은 결과가 나오는지 묻는다.

### 체크포인트 23: 검증 기준 되돌아본다

저장 파일의 신뢰도를 높이기 위해, 레슨 10 통합 프로젝트: 자동화 리포트 만들기에서는 검증 기준을/를 되돌아본다는 과정을 별도의 단계로 둔다. 이때 학생은 화면에 보이는 값 하나보다 어떤 입력에서 어떤 변환을 거쳐 결과가 나왔는지 설명해야 한다. 강사는 변수명, 출력 형태, 저장 여부를 함께 확인하며 다음 실행에서도 같은 결과가 나오는지 묻는다.
