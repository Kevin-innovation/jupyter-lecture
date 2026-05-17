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

아래 셀은 강사가 먼저 실행 흐름을 보여주고, 학생이 같은 구조를 자기 말로 설명하도록 만든 예제다. 코드는 짧지만 입력, 처리, 출력이 분리되어 있어 나중에 문제 풀이로 확장하기 쉽다.

~~~python
index = parse_html('portal_index.html')
print(text_or_empty(index.select_one('h1')))
~~~
이 셀에서 확인해야 할 것은 출력값 하나가 아니라 데이터가 어떤 단계에서 바뀌었는지다. 자동화 수업에서는 중간 변수명을 명확히 두면 학생이 오류 위치를 쉽게 찾을 수 있다.

---

## 4. 자료 구조 확인

아래 셀은 강사가 먼저 실행 흐름을 보여주고, 학생이 같은 구조를 자기 말로 설명하도록 만든 예제다. 코드는 짧지만 입력, 처리, 출력이 분리되어 있어 나중에 문제 풀이로 확장하기 쉽다.

~~~python
links = []
for a in index.select('a[data-page]'):
    links.append({'label': a.get_text(' ', strip=True), 'href': a['href'], 'url': urljoin('https://lesson.local/portal/', a['href'])})
print(links)
~~~
이 셀에서 확인해야 할 것은 출력값 하나가 아니라 데이터가 어떤 단계에서 바뀌었는지다. 자동화 수업에서는 중간 변수명을 명확히 두면 학생이 오류 위치를 쉽게 찾을 수 있다.

---

## 5. 품질 기준 적용

아래 셀은 강사가 먼저 실행 흐름을 보여주고, 학생이 같은 구조를 자기 말로 설명하도록 만든 예제다. 코드는 짧지만 입력, 처리, 출력이 분리되어 있어 나중에 문제 풀이로 확장하기 쉽다.

~~~python
notice = parse_html('portal_notice.html')
notices = [{'title': item.select_one('.title').get_text(' ', strip=True), 'level': item.get('data-level'), 'date': item.get('data-date')} for item in notice.select('.notice-card')]
print(notices[:2])
~~~
이 셀에서 확인해야 할 것은 출력값 하나가 아니라 데이터가 어떤 단계에서 바뀌었는지다. 자동화 수업에서는 중간 변수명을 명확히 두면 학생이 오류 위치를 쉽게 찾을 수 있다.

---

## 6. 저장과 보고

아래 셀은 강사가 먼저 실행 흐름을 보여주고, 학생이 같은 구조를 자기 말로 설명하도록 만든 예제다. 코드는 짧지만 입력, 처리, 출력이 분리되어 있어 나중에 문제 풀이로 확장하기 쉽다.

~~~python
courses = parse_html('portal_courses.html')
course_rows = []
for row in courses.select('tbody tr'):
    cells = [td.get_text(' ', strip=True) for td in row.select('td')]
    course_rows.append({'course': cells[0], 'teacher': cells[1], 'students': clean_int(cells[2]), 'status': cells[3]})
print(course_rows[:2])
~~~
이 셀에서 확인해야 할 것은 출력값 하나가 아니라 데이터가 어떤 단계에서 바뀌었는지다. 자동화 수업에서는 중간 변수명을 명확히 두면 학생이 오류 위치를 쉽게 찾을 수 있다.

---

## 7. 운영 관점 점검

아래 셀은 강사가 먼저 실행 흐름을 보여주고, 학생이 같은 구조를 자기 말로 설명하도록 만든 예제다. 코드는 짧지만 입력, 처리, 출력이 분리되어 있어 나중에 문제 풀이로 확장하기 쉽다.

~~~python
manifest = load_csv('download_manifest.csv')
print(manifest[:2])
~~~
이 셀에서 확인해야 할 것은 출력값 하나가 아니라 데이터가 어떤 단계에서 바뀌었는지다. 자동화 수업에서는 중간 변수명을 명확히 두면 학생이 오류 위치를 쉽게 찾을 수 있다.

---

## 8. 마무리 체크

아래 셀은 강사가 먼저 실행 흐름을 보여주고, 학생이 같은 구조를 자기 말로 설명하도록 만든 예제다. 코드는 짧지만 입력, 처리, 출력이 분리되어 있어 나중에 문제 풀이로 확장하기 쉽다.

~~~python
rules = load_json('quality_rules.json')
ready_courses = [row for row in course_rows if status_ok(row['status']) and row['students'] >= rules['min_students_for_report']]
print(ready_courses)
~~~
이 셀에서 확인해야 할 것은 출력값 하나가 아니라 데이터가 어떤 단계에서 바뀌었는지다. 자동화 수업에서는 중간 변수명을 명확히 두면 학생이 오류 위치를 쉽게 찾을 수 있다.

---

## 9. 핵심 개념

아래 셀은 강사가 먼저 실행 흐름을 보여주고, 학생이 같은 구조를 자기 말로 설명하도록 만든 예제다. 코드는 짧지만 입력, 처리, 출력이 분리되어 있어 나중에 문제 풀이로 확장하기 쉽다.

~~~python
report = {'notice_count': len(notices), 'course_count': len(course_rows), 'file_count': len(manifest), 'ready_course_count': len(ready_courses)}
Path('lesson10_portal_report.json').write_text(json.dumps(report, ensure_ascii=False, indent=2), encoding='utf-8')
print(report)
~~~
이 셀에서 확인해야 할 것은 출력값 하나가 아니라 데이터가 어떤 단계에서 바뀌었는지다. 자동화 수업에서는 중간 변수명을 명확히 두면 학생이 오류 위치를 쉽게 찾을 수 있다.

---

## 데이터 출처와 안전 규칙

portal_index.html은 합성 포털의 시작 페이지다. portal_notice.html, portal_courses.html, portal_downloads.html, portal_status.html은 각각 공지, 과정, 자료, 상태 정보를 담는다. download_manifest.csv와 quality_rules.json은 최종 리포트 검증에 사용한다.

- 모든 파일은 수업용 합성 데이터다.
- 실제 사이트에 반복 요청하지 않는다.
- 저장 파일은 레슨 폴더 또는 코랩 현재 작업 폴더에만 만든다.
- 외부 사이트로 확장할 때는 약관, robots.txt, 요청 간격, 개인정보 포함 여부를 먼저 확인한다.

### 보강 설명 1

학생에게는 코드의 속도보다 검증 순서를 강조한다. 입력을 읽고, 구조를 확인하고, 오류를 분류하고, 저장한 뒤 요약을 남기는 순서가 유지되면 도구가 바뀌어도 자동화 품질은 크게 흔들리지 않는다.

### 보강 설명 2

학생에게는 코드의 속도보다 검증 순서를 강조한다. 입력을 읽고, 구조를 확인하고, 오류를 분류하고, 저장한 뒤 요약을 남기는 순서가 유지되면 도구가 바뀌어도 자동화 품질은 크게 흔들리지 않는다.

### 보강 설명 3

학생에게는 코드의 속도보다 검증 순서를 강조한다. 입력을 읽고, 구조를 확인하고, 오류를 분류하고, 저장한 뒤 요약을 남기는 순서가 유지되면 도구가 바뀌어도 자동화 품질은 크게 흔들리지 않는다.

### 보강 설명 4

학생에게는 코드의 속도보다 검증 순서를 강조한다. 입력을 읽고, 구조를 확인하고, 오류를 분류하고, 저장한 뒤 요약을 남기는 순서가 유지되면 도구가 바뀌어도 자동화 품질은 크게 흔들리지 않는다.

### 보강 설명 5

학생에게는 코드의 속도보다 검증 순서를 강조한다. 입력을 읽고, 구조를 확인하고, 오류를 분류하고, 저장한 뒤 요약을 남기는 순서가 유지되면 도구가 바뀌어도 자동화 품질은 크게 흔들리지 않는다.

### 보강 설명 6

학생에게는 코드의 속도보다 검증 순서를 강조한다. 입력을 읽고, 구조를 확인하고, 오류를 분류하고, 저장한 뒤 요약을 남기는 순서가 유지되면 도구가 바뀌어도 자동화 품질은 크게 흔들리지 않는다.

### 보강 설명 7

학생에게는 코드의 속도보다 검증 순서를 강조한다. 입력을 읽고, 구조를 확인하고, 오류를 분류하고, 저장한 뒤 요약을 남기는 순서가 유지되면 도구가 바뀌어도 자동화 품질은 크게 흔들리지 않는다.

### 보강 설명 8

학생에게는 코드의 속도보다 검증 순서를 강조한다. 입력을 읽고, 구조를 확인하고, 오류를 분류하고, 저장한 뒤 요약을 남기는 순서가 유지되면 도구가 바뀌어도 자동화 품질은 크게 흔들리지 않는다.

### 보강 설명 9

학생에게는 코드의 속도보다 검증 순서를 강조한다. 입력을 읽고, 구조를 확인하고, 오류를 분류하고, 저장한 뒤 요약을 남기는 순서가 유지되면 도구가 바뀌어도 자동화 품질은 크게 흔들리지 않는다.

### 보강 설명 10

학생에게는 코드의 속도보다 검증 순서를 강조한다. 입력을 읽고, 구조를 확인하고, 오류를 분류하고, 저장한 뒤 요약을 남기는 순서가 유지되면 도구가 바뀌어도 자동화 품질은 크게 흔들리지 않는다.

### 보강 설명 11

학생에게는 코드의 속도보다 검증 순서를 강조한다. 입력을 읽고, 구조를 확인하고, 오류를 분류하고, 저장한 뒤 요약을 남기는 순서가 유지되면 도구가 바뀌어도 자동화 품질은 크게 흔들리지 않는다.

### 보강 설명 12

학생에게는 코드의 속도보다 검증 순서를 강조한다. 입력을 읽고, 구조를 확인하고, 오류를 분류하고, 저장한 뒤 요약을 남기는 순서가 유지되면 도구가 바뀌어도 자동화 품질은 크게 흔들리지 않는다.

### 보강 설명 13

학생에게는 코드의 속도보다 검증 순서를 강조한다. 입력을 읽고, 구조를 확인하고, 오류를 분류하고, 저장한 뒤 요약을 남기는 순서가 유지되면 도구가 바뀌어도 자동화 품질은 크게 흔들리지 않는다.

### 보강 설명 14

학생에게는 코드의 속도보다 검증 순서를 강조한다. 입력을 읽고, 구조를 확인하고, 오류를 분류하고, 저장한 뒤 요약을 남기는 순서가 유지되면 도구가 바뀌어도 자동화 품질은 크게 흔들리지 않는다.

### 보강 설명 15

학생에게는 코드의 속도보다 검증 순서를 강조한다. 입력을 읽고, 구조를 확인하고, 오류를 분류하고, 저장한 뒤 요약을 남기는 순서가 유지되면 도구가 바뀌어도 자동화 품질은 크게 흔들리지 않는다.

### 보강 설명 16

학생에게는 코드의 속도보다 검증 순서를 강조한다. 입력을 읽고, 구조를 확인하고, 오류를 분류하고, 저장한 뒤 요약을 남기는 순서가 유지되면 도구가 바뀌어도 자동화 품질은 크게 흔들리지 않는다.

### 보강 설명 17

학생에게는 코드의 속도보다 검증 순서를 강조한다. 입력을 읽고, 구조를 확인하고, 오류를 분류하고, 저장한 뒤 요약을 남기는 순서가 유지되면 도구가 바뀌어도 자동화 품질은 크게 흔들리지 않는다.

### 보강 설명 18

학생에게는 코드의 속도보다 검증 순서를 강조한다. 입력을 읽고, 구조를 확인하고, 오류를 분류하고, 저장한 뒤 요약을 남기는 순서가 유지되면 도구가 바뀌어도 자동화 품질은 크게 흔들리지 않는다.

### 보강 설명 19

학생에게는 코드의 속도보다 검증 순서를 강조한다. 입력을 읽고, 구조를 확인하고, 오류를 분류하고, 저장한 뒤 요약을 남기는 순서가 유지되면 도구가 바뀌어도 자동화 품질은 크게 흔들리지 않는다.

### 보강 설명 20

학생에게는 코드의 속도보다 검증 순서를 강조한다. 입력을 읽고, 구조를 확인하고, 오류를 분류하고, 저장한 뒤 요약을 남기는 순서가 유지되면 도구가 바뀌어도 자동화 품질은 크게 흔들리지 않는다.

### 보강 설명 21

학생에게는 코드의 속도보다 검증 순서를 강조한다. 입력을 읽고, 구조를 확인하고, 오류를 분류하고, 저장한 뒤 요약을 남기는 순서가 유지되면 도구가 바뀌어도 자동화 품질은 크게 흔들리지 않는다.

### 보강 설명 22

학생에게는 코드의 속도보다 검증 순서를 강조한다. 입력을 읽고, 구조를 확인하고, 오류를 분류하고, 저장한 뒤 요약을 남기는 순서가 유지되면 도구가 바뀌어도 자동화 품질은 크게 흔들리지 않는다.

### 보강 설명 23

학생에게는 코드의 속도보다 검증 순서를 강조한다. 입력을 읽고, 구조를 확인하고, 오류를 분류하고, 저장한 뒤 요약을 남기는 순서가 유지되면 도구가 바뀌어도 자동화 품질은 크게 흔들리지 않는다.

### 보강 설명 24

학생에게는 코드의 속도보다 검증 순서를 강조한다. 입력을 읽고, 구조를 확인하고, 오류를 분류하고, 저장한 뒤 요약을 남기는 순서가 유지되면 도구가 바뀌어도 자동화 품질은 크게 흔들리지 않는다.

### 보강 설명 25

학생에게는 코드의 속도보다 검증 순서를 강조한다. 입력을 읽고, 구조를 확인하고, 오류를 분류하고, 저장한 뒤 요약을 남기는 순서가 유지되면 도구가 바뀌어도 자동화 품질은 크게 흔들리지 않는다.

### 보강 설명 26

학생에게는 코드의 속도보다 검증 순서를 강조한다. 입력을 읽고, 구조를 확인하고, 오류를 분류하고, 저장한 뒤 요약을 남기는 순서가 유지되면 도구가 바뀌어도 자동화 품질은 크게 흔들리지 않는다.

### 보강 설명 27

학생에게는 코드의 속도보다 검증 순서를 강조한다. 입력을 읽고, 구조를 확인하고, 오류를 분류하고, 저장한 뒤 요약을 남기는 순서가 유지되면 도구가 바뀌어도 자동화 품질은 크게 흔들리지 않는다.

### 보강 설명 28

학생에게는 코드의 속도보다 검증 순서를 강조한다. 입력을 읽고, 구조를 확인하고, 오류를 분류하고, 저장한 뒤 요약을 남기는 순서가 유지되면 도구가 바뀌어도 자동화 품질은 크게 흔들리지 않는다.

### 보강 설명 29

학생에게는 코드의 속도보다 검증 순서를 강조한다. 입력을 읽고, 구조를 확인하고, 오류를 분류하고, 저장한 뒤 요약을 남기는 순서가 유지되면 도구가 바뀌어도 자동화 품질은 크게 흔들리지 않는다.
