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

# 레슨 10 — 실습 문제

[![Open in Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/Kevin-innovation/jupyter-lecture/blob/main/python-web-automation/lectures/10/%5B%ED%95%99%EC%83%9D%EC%9A%A9%5D%20%EB%A0%88%EC%8A%A8%2010%20%E2%80%94%20%ED%86%B5%ED%95%A9%20%ED%94%84%EB%A1%9C%EC%A0%9D%ED%8A%B8%3A%20%EC%9E%90%EB%8F%99%ED%99%94%20%EB%A6%AC%ED%8F%AC%ED%8A%B8%20%EB%A7%8C%EB%93%A4%EA%B8%B0.ipynb)

> 코랩에서 문제를 풀려면 우측 상단 또는 아래의 **Open in Colab** 버튼을 클릭한다. 빈칸은 정답을 복사하지 말고 데이터 구조를 확인한 뒤 채운다.


통합 프로젝트: 자동화 리포트 만들기 레슨의 학생용 문제 노트북이다. 강의 노트북을 먼저 실행한 뒤 빈칸을 직접 채운다.

## 통과 기준

- 총 15문제 중 12문제 이상 정상 출력이면 통과.
- 문제 1~5는 구조 확인, 6~10은 응용 처리, 11~15는 저장과 운영 요약이다.
- 정답값은 적지 않는다. 출력 형태와 fixture 구조를 보고 직접 판단한다.

## 0. 환경 셀

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

## 문제 1 — 포털 제목 읽기

지시된 값을 코드로 추출하거나 상태를 변경한다. 실행 결과는 한 줄 출력, 리스트, 딕셔너리, 저장 파일 생성 중 하나다.

**기대 결과 형태**: 요구한 값이 출력되거나 지정한 파일이 생성된다.

**빈칸 힌트**: 시작 HTML 파일과 제목 selector를 채운다. 완성 코드를 그대로 따라 쓰지 말고 `____` 부분만 스스로 판단한다.

~~~python
index = parse_html('____')
print(text_or_empty(index.select_one('____')))
~~~

---

## 문제 2 — 목차 링크 수집

지시된 값을 코드로 추출하거나 상태를 변경한다. 실행 결과는 한 줄 출력, 리스트, 딕셔너리, 저장 파일 생성 중 하나다.

**기대 결과 형태**: 요구한 값이 출력되거나 지정한 파일이 생성된다.

**빈칸 힌트**: 목차 링크에는 data-page 속성이 있다. 완성 코드를 그대로 따라 쓰지 말고 `____` 부분만 스스로 판단한다.

~~~python
links = []
for a in index.select('____'):
    links.append({'label': a.get_text(' ', strip=True), 'href': a['href']})
print(links)
~~~

---

## 문제 3 — 상대 URL을 절대 URL로 바꾸기

지시된 값을 코드로 추출하거나 상태를 변경한다. 실행 결과는 한 줄 출력, 리스트, 딕셔너리, 저장 파일 생성 중 하나다.

**기대 결과 형태**: 요구한 값이 출력되거나 지정한 파일이 생성된다.

**빈칸 힌트**: href 값을 사용한다. 완성 코드를 그대로 따라 쓰지 말고 `____` 부분만 스스로 판단한다.

~~~python
base = 'https://lesson.local/portal/'
absolute = [urljoin(base, item['____']) for item in links]
print(absolute)
~~~

---

## 문제 4 — 공지 카드 추출

지시된 값을 코드로 추출하거나 상태를 변경한다. 실행 결과는 한 줄 출력, 리스트, 딕셔너리, 저장 파일 생성 중 하나다.

**기대 결과 형태**: 요구한 값이 출력되거나 지정한 파일이 생성된다.

**빈칸 힌트**: 공지 파일과 카드 selector를 찾는다. 완성 코드를 그대로 따라 쓰지 말고 `____` 부분만 스스로 판단한다.

~~~python
notice = parse_html('____')
notice_cards = notice.select('____')
print(len(notice_cards))
~~~

---

## 문제 5 — 공지 제목과 레벨 정리

지시된 값을 코드로 추출하거나 상태를 변경한다. 실행 결과는 한 줄 출력, 리스트, 딕셔너리, 저장 파일 생성 중 하나다.

**기대 결과 형태**: 요구한 값이 출력되거나 지정한 파일이 생성된다.

**빈칸 힌트**: 제목 class와 data 속성 이름을 채운다. 완성 코드를 그대로 따라 쓰지 말고 `____` 부분만 스스로 판단한다.

~~~python
notices = []
for card in notice_cards:
    notices.append({'title': text_or_empty(card.select_one('____')), 'level': card.get('____'), 'date': card.get('data-date')})
print(notices[:3])
~~~

---

## 문제 6 — 과정 표 행 읽기

지시된 값을 코드로 추출하거나 상태를 변경한다. 실행 결과는 한 줄 출력, 리스트, 딕셔너리, 저장 파일 생성 중 하나다.

**기대 결과 형태**: 요구한 값이 출력되거나 지정한 파일이 생성된다.

**빈칸 힌트**: 과정 HTML과 표 셀 selector를 넣는다. 완성 코드를 그대로 따라 쓰지 말고 `____` 부분만 스스로 판단한다.

~~~python
courses = parse_html('____')
course_rows = []
for row in courses.select('tbody tr'):
    cells = [td.get_text(' ', strip=True) for td in row.select('____')]
    course_rows.append({'course': cells[0], 'teacher': cells[1], 'students': clean_int(cells[2]), 'status': cells[3]})
print(course_rows[:2])
~~~

---

## 문제 7 — 다운로드 manifest 읽기

지시된 값을 코드로 추출하거나 상태를 변경한다. 실행 결과는 한 줄 출력, 리스트, 딕셔너리, 저장 파일 생성 중 하나다.

**기대 결과 형태**: 요구한 값이 출력되거나 지정한 파일이 생성된다.

**빈칸 힌트**: 자료 목록 CSV와 파일명 컬럼을 확인한다. 완성 코드를 그대로 따라 쓰지 말고 `____` 부분만 스스로 판단한다.

~~~python
manifest = load_csv('____')
print(manifest[0]['____'])
~~~

---

## 문제 8 — 상태 페이지 metric 추출

지시된 값을 코드로 추출하거나 상태를 변경한다. 실행 결과는 한 줄 출력, 리스트, 딕셔너리, 저장 파일 생성 중 하나다.

**기대 결과 형태**: 요구한 값이 출력되거나 지정한 파일이 생성된다.

**빈칸 힌트**: 상태 HTML과 metric selector를 채운다. 완성 코드를 그대로 따라 쓰지 말고 `____` 부분만 스스로 판단한다.

~~~python
status_page = parse_html('____')
metrics = {item.get('data-name'): clean_int(item.get_text(' ', strip=True)) for item in status_page.select('____')}
print(metrics)
~~~

---

## 문제 9 — 품질 규칙 읽기

지시된 값을 코드로 추출하거나 상태를 변경한다. 실행 결과는 한 줄 출력, 리스트, 딕셔너리, 저장 파일 생성 중 하나다.

**기대 결과 형태**: 요구한 값이 출력되거나 지정한 파일이 생성된다.

**빈칸 힌트**: 규칙 파일과 최소 학생 수 키를 찾는다. 완성 코드를 그대로 따라 쓰지 말고 `____` 부분만 스스로 판단한다.

~~~python
rules = load_json('____')
print(rules['____'])
~~~

---

## 문제 10 — 리포트 대상 과정 필터링

지시된 값을 코드로 추출하거나 상태를 변경한다. 실행 결과는 한 줄 출력, 리스트, 딕셔너리, 저장 파일 생성 중 하나다.

**기대 결과 형태**: 요구한 값이 출력되거나 지정한 파일이 생성된다.

**빈칸 힌트**: 최소 학생 수 키를 넣는다. 완성 코드를 그대로 따라 쓰지 말고 `____` 부분만 스스로 판단한다.

~~~python
ready_courses = []
for row in course_rows:
    if status_ok(row['status']) and row['students'] >= rules['____']:
        ready_courses.append(row)
print(ready_courses)
~~~

---

## 문제 11 — 자료 파일 중복 키 만들기

지시된 값을 코드로 추출하거나 상태를 변경한다. 실행 결과는 한 줄 출력, 리스트, 딕셔너리, 저장 파일 생성 중 하나다.

**기대 결과 형태**: 요구한 값이 출력되거나 지정한 파일이 생성된다.

**빈칸 힌트**: 파일명 컬럼을 넣는다. 완성 코드를 그대로 따라 쓰지 말고 `____` 부분만 스스로 판단한다.

~~~python
keys = [make_key(row['course'], row['____']) for row in manifest]
print(len(keys), len(set(keys)))
~~~

---

## 문제 12 — 통합 CSV 행 만들기

지시된 값을 코드로 추출하거나 상태를 변경한다. 실행 결과는 한 줄 출력, 리스트, 딕셔너리, 저장 파일 생성 중 하나다.

**기대 결과 형태**: 요구한 값이 출력되거나 지정한 파일이 생성된다.

**빈칸 힌트**: 과정별 자료 목록 변수를 넣는다. 완성 코드를 그대로 따라 쓰지 말고 `____` 부분만 스스로 판단한다.

~~~python
report_rows = []
for row in ready_courses:
    files = [item for item in manifest if item['course'] == row['course']]
    report_rows.append({'course': row['course'], 'teacher': row['teacher'], 'students': row['students'], 'file_count': len(____), 'status': row['status']})
print(report_rows)
~~~

---

## 문제 13 — CSV와 JSON 저장

지시된 값을 코드로 추출하거나 상태를 변경한다. 실행 결과는 한 줄 출력, 리스트, 딕셔너리, 저장 파일 생성 중 하나다.

**기대 결과 형태**: 요구한 값이 출력되거나 지정한 파일이 생성된다.

**빈칸 힌트**: 전체 자료 목록 개수를 넣는다. 완성 코드를 그대로 따라 쓰지 말고 `____` 부분만 스스로 판단한다.

~~~python
write_csv('lesson10_portal_report.csv', report_rows)
summary = {'notice_count': len(notices), 'course_count': len(course_rows), 'ready_course_count': len(report_rows), 'file_count': len(____)}
Path('lesson10_portal_report.json').write_text(json.dumps(summary, ensure_ascii=False, indent=2), encoding='utf-8')
print(summary)
~~~

---

## 문제 14 — SQLite 저장과 조회

지시된 값을 코드로 추출하거나 상태를 변경한다. 실행 결과는 한 줄 출력, 리스트, 딕셔너리, 저장 파일 생성 중 하나다.

**기대 결과 형태**: 요구한 값이 출력되거나 지정한 파일이 생성된다.

**빈칸 힌트**: 저장할 통합 행 목록을 넣는다. 완성 코드를 그대로 따라 쓰지 말고 `____` 부분만 스스로 판단한다.

~~~python
conn = sqlite3.connect('lesson10_portal.db')
conn.execute('drop table if exists report')
conn.execute('create table report(course text, teacher text, students integer, file_count integer, status text)')
conn.executemany('insert into report values(:course, :teacher, :students, :file_count, :status)', ____)
print(conn.execute('select course, file_count from report order by students desc').fetchall())
conn.close()
~~~

---

## 문제 15 — 운영 메모 3문장 작성

지시된 값을 코드로 추출하거나 상태를 변경한다. 실행 결과는 한 줄 출력, 리스트, 딕셔너리, 저장 파일 생성 중 하나다.

**기대 결과 형태**: 요구한 값이 출력되거나 지정한 파일이 생성된다.

**빈칸 힌트**: 작성한 문장 리스트를 출력한다. 완성 코드를 그대로 따라 쓰지 말고 `____` 부분만 스스로 판단한다.

~~~python
memo = [
    f"공지 {len(notices)}건, 과정 {len(course_rows)}건을 확인했습니다.",
    f"리포트 대상 과정은 {len(report_rows)}건이며 자료 파일은 {len(manifest)}개입니다.",
    f"최근 실행 수는 {metrics.get('runs', 0)}회이고 오류 수는 {metrics.get('errors', 0)}회입니다.",
]
print('\n'.join(____))
~~~


## 문제 풀이 전 데이터 점검

문제를 풀기 전에 data/ 폴더의 파일 이름을 먼저 확인한다. 이번 레슨은 하나의 HTML만 읽는 문제가 아니라 포털 시작 페이지, 공지 카드, 과정 표, 다운로드 manifest, 상태 metric, 품질 규칙을 순서대로 연결하는 프로젝트다. 빈칸을 채울 때는 이전 문제에서 만든 변수를 그대로 이어 써야 하며, 중간 변수를 새로 만들었다면 다음 문제에서 같은 이름을 사용해야 한다.

- portal_index.html: 포털 제목과 하위 페이지 링크를 확인한다.
- portal_notice.html: 공지 카드의 제목, 레벨, 날짜를 읽는다.
- portal_courses.html: 과정명, 담당자, 학생 수, 상태를 표에서 읽는다.
- download_manifest.csv: 과정별 자료 파일과 파일 종류를 확인한다.
- portal_status.html: 실행 수, 제출 수, 오류 수 metric을 읽는다.
- quality_rules.json: 리포트 대상 과정을 고르는 기준을 읽는다.

## 제출 전 확인

1. 15문제 중 12문제 이상이 오류 없이 실행되어야 한다.
2. lesson10_portal_report.csv 또는 lesson10_portal_report.json 중 하나 이상이 생성되어야 한다.
3. 운영 메모 3문장에는 입력 개수, 리포트 대상 개수, 상태 metric 중 하나 이상이 들어가야 한다.
4. 외부 사이트 URL로 직접 요청을 보내지 않고 제공된 fixture만 사용해야 한다.
