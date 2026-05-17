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

# 레슨 10 — 실습 문제 정답지

> 🔒 교사·관리자 전용. 학생에게 배포 금지.

통합 프로젝트: 자동화 리포트 만들기 실습 문제의 모범 답안이다. 출력값만 보지 말고 검증 기준, 저장 구조, 로그를 함께 확인한다.

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

## 문제 1 정답 — 포털 제목 읽기

~~~python
index = parse_html('portal_index.html')
print(text_or_empty(index.select_one('h1')))
~~~

### 왜 이 코드가 정답인지

이 답안은 문제에서 요구한 입력 구조를 먼저 읽고, 필요한 컬럼이나 selector만 선택한 뒤 결과를 명시적인 변수에 저장한다. `포털 제목 읽기` 단계에서 중요한 점은 출력 하나를 맞히는 것이 아니라 이후 문제에서 재사용할 수 있는 안정적인 중간 결과를 남기는 것이다. 빈칸에는 파일명, 컬럼명, 키 이름, 반복 제어 중 핵심 위치만 남겼기 때문에 학생이 데이터 구조를 이해해야 완성할 수 있다. 자주 보이는 오답은 컬럼명을 추측하거나, 중복과 실패 상태를 처리하지 않고 바로 저장하는 방식이다.

---

## 문제 2 정답 — 목차 링크 수집

~~~python
links = []
for a in index.select('a[data-page]'):
    links.append({'label': a.get_text(' ', strip=True), 'href': a['href']})
print(links)
~~~

### 왜 이 코드가 정답인지

이 답안은 문제에서 요구한 입력 구조를 먼저 읽고, 필요한 컬럼이나 selector만 선택한 뒤 결과를 명시적인 변수에 저장한다. `목차 링크 수집` 단계에서 중요한 점은 출력 하나를 맞히는 것이 아니라 이후 문제에서 재사용할 수 있는 안정적인 중간 결과를 남기는 것이다. 빈칸에는 파일명, 컬럼명, 키 이름, 반복 제어 중 핵심 위치만 남겼기 때문에 학생이 데이터 구조를 이해해야 완성할 수 있다. 자주 보이는 오답은 컬럼명을 추측하거나, 중복과 실패 상태를 처리하지 않고 바로 저장하는 방식이다.

---

## 문제 3 정답 — 상대 URL을 절대 URL로 바꾸기

~~~python
base = 'https://lesson.local/portal/'
absolute = [urljoin(base, item['href']) for item in links]
print(absolute)
~~~

### 왜 이 코드가 정답인지

이 답안은 문제에서 요구한 입력 구조를 먼저 읽고, 필요한 컬럼이나 selector만 선택한 뒤 결과를 명시적인 변수에 저장한다. `상대 URL을 절대 URL로 바꾸기` 단계에서 중요한 점은 출력 하나를 맞히는 것이 아니라 이후 문제에서 재사용할 수 있는 안정적인 중간 결과를 남기는 것이다. 빈칸에는 파일명, 컬럼명, 키 이름, 반복 제어 중 핵심 위치만 남겼기 때문에 학생이 데이터 구조를 이해해야 완성할 수 있다. 자주 보이는 오답은 컬럼명을 추측하거나, 중복과 실패 상태를 처리하지 않고 바로 저장하는 방식이다.

---

## 문제 4 정답 — 공지 카드 추출

~~~python
notice = parse_html('portal_notice.html')
notice_cards = notice.select('.notice-card')
print(len(notice_cards))
~~~

### 왜 이 코드가 정답인지

이 답안은 문제에서 요구한 입력 구조를 먼저 읽고, 필요한 컬럼이나 selector만 선택한 뒤 결과를 명시적인 변수에 저장한다. `공지 카드 추출` 단계에서 중요한 점은 출력 하나를 맞히는 것이 아니라 이후 문제에서 재사용할 수 있는 안정적인 중간 결과를 남기는 것이다. 빈칸에는 파일명, 컬럼명, 키 이름, 반복 제어 중 핵심 위치만 남겼기 때문에 학생이 데이터 구조를 이해해야 완성할 수 있다. 자주 보이는 오답은 컬럼명을 추측하거나, 중복과 실패 상태를 처리하지 않고 바로 저장하는 방식이다.

---

## 문제 5 정답 — 공지 제목과 레벨 정리

~~~python
notices = []
for card in notice_cards:
    notices.append({'title': text_or_empty(card.select_one('.title')), 'level': card.get('data-level'), 'date': card.get('data-date')})
print(notices[:3])
~~~

### 왜 이 코드가 정답인지

이 답안은 문제에서 요구한 입력 구조를 먼저 읽고, 필요한 컬럼이나 selector만 선택한 뒤 결과를 명시적인 변수에 저장한다. `공지 제목과 레벨 정리` 단계에서 중요한 점은 출력 하나를 맞히는 것이 아니라 이후 문제에서 재사용할 수 있는 안정적인 중간 결과를 남기는 것이다. 빈칸에는 파일명, 컬럼명, 키 이름, 반복 제어 중 핵심 위치만 남겼기 때문에 학생이 데이터 구조를 이해해야 완성할 수 있다. 자주 보이는 오답은 컬럼명을 추측하거나, 중복과 실패 상태를 처리하지 않고 바로 저장하는 방식이다.

---

## 문제 6 정답 — 과정 표 행 읽기

~~~python
courses = parse_html('portal_courses.html')
course_rows = []
for row in courses.select('tbody tr'):
    cells = [td.get_text(' ', strip=True) for td in row.select('td')]
    course_rows.append({'course': cells[0], 'teacher': cells[1], 'students': clean_int(cells[2]), 'status': cells[3]})
print(course_rows[:2])
~~~

### 왜 이 코드가 정답인지

이 답안은 문제에서 요구한 입력 구조를 먼저 읽고, 필요한 컬럼이나 selector만 선택한 뒤 결과를 명시적인 변수에 저장한다. `과정 표 행 읽기` 단계에서 중요한 점은 출력 하나를 맞히는 것이 아니라 이후 문제에서 재사용할 수 있는 안정적인 중간 결과를 남기는 것이다. 빈칸에는 파일명, 컬럼명, 키 이름, 반복 제어 중 핵심 위치만 남겼기 때문에 학생이 데이터 구조를 이해해야 완성할 수 있다. 자주 보이는 오답은 컬럼명을 추측하거나, 중복과 실패 상태를 처리하지 않고 바로 저장하는 방식이다.

---

## 문제 7 정답 — 다운로드 manifest 읽기

~~~python
manifest = load_csv('download_manifest.csv')
print(manifest[0]['filename'])
~~~

### 왜 이 코드가 정답인지

이 답안은 문제에서 요구한 입력 구조를 먼저 읽고, 필요한 컬럼이나 selector만 선택한 뒤 결과를 명시적인 변수에 저장한다. `다운로드 manifest 읽기` 단계에서 중요한 점은 출력 하나를 맞히는 것이 아니라 이후 문제에서 재사용할 수 있는 안정적인 중간 결과를 남기는 것이다. 빈칸에는 파일명, 컬럼명, 키 이름, 반복 제어 중 핵심 위치만 남겼기 때문에 학생이 데이터 구조를 이해해야 완성할 수 있다. 자주 보이는 오답은 컬럼명을 추측하거나, 중복과 실패 상태를 처리하지 않고 바로 저장하는 방식이다.

---

## 문제 8 정답 — 상태 페이지 metric 추출

~~~python
status_page = parse_html('portal_status.html')
metrics = {item.get('data-name'): clean_int(item.get_text(' ', strip=True)) for item in status_page.select('.metric')}
print(metrics)
~~~

### 왜 이 코드가 정답인지

이 답안은 문제에서 요구한 입력 구조를 먼저 읽고, 필요한 컬럼이나 selector만 선택한 뒤 결과를 명시적인 변수에 저장한다. `상태 페이지 metric 추출` 단계에서 중요한 점은 출력 하나를 맞히는 것이 아니라 이후 문제에서 재사용할 수 있는 안정적인 중간 결과를 남기는 것이다. 빈칸에는 파일명, 컬럼명, 키 이름, 반복 제어 중 핵심 위치만 남겼기 때문에 학생이 데이터 구조를 이해해야 완성할 수 있다. 자주 보이는 오답은 컬럼명을 추측하거나, 중복과 실패 상태를 처리하지 않고 바로 저장하는 방식이다.

---

## 문제 9 정답 — 품질 규칙 읽기

~~~python
rules = load_json('quality_rules.json')
print(rules['min_students_for_report'])
~~~

### 왜 이 코드가 정답인지

이 답안은 문제에서 요구한 입력 구조를 먼저 읽고, 필요한 컬럼이나 selector만 선택한 뒤 결과를 명시적인 변수에 저장한다. `품질 규칙 읽기` 단계에서 중요한 점은 출력 하나를 맞히는 것이 아니라 이후 문제에서 재사용할 수 있는 안정적인 중간 결과를 남기는 것이다. 빈칸에는 파일명, 컬럼명, 키 이름, 반복 제어 중 핵심 위치만 남겼기 때문에 학생이 데이터 구조를 이해해야 완성할 수 있다. 자주 보이는 오답은 컬럼명을 추측하거나, 중복과 실패 상태를 처리하지 않고 바로 저장하는 방식이다.

---

## 문제 10 정답 — 리포트 대상 과정 필터링

~~~python
ready_courses = []
for row in course_rows:
    if status_ok(row['status']) and row['students'] >= rules['min_students_for_report']:
        ready_courses.append(row)
print(ready_courses)
~~~

### 왜 이 코드가 정답인지

이 답안은 문제에서 요구한 입력 구조를 먼저 읽고, 필요한 컬럼이나 selector만 선택한 뒤 결과를 명시적인 변수에 저장한다. `리포트 대상 과정 필터링` 단계에서 중요한 점은 출력 하나를 맞히는 것이 아니라 이후 문제에서 재사용할 수 있는 안정적인 중간 결과를 남기는 것이다. 빈칸에는 파일명, 컬럼명, 키 이름, 반복 제어 중 핵심 위치만 남겼기 때문에 학생이 데이터 구조를 이해해야 완성할 수 있다. 자주 보이는 오답은 컬럼명을 추측하거나, 중복과 실패 상태를 처리하지 않고 바로 저장하는 방식이다.

---

## 문제 11 정답 — 자료 파일 중복 키 만들기

~~~python
keys = [make_key(row['course'], row['filename']) for row in manifest]
print(len(keys), len(set(keys)))
~~~

### 왜 이 코드가 정답인지

이 답안은 문제에서 요구한 입력 구조를 먼저 읽고, 필요한 컬럼이나 selector만 선택한 뒤 결과를 명시적인 변수에 저장한다. `자료 파일 중복 키 만들기` 단계에서 중요한 점은 출력 하나를 맞히는 것이 아니라 이후 문제에서 재사용할 수 있는 안정적인 중간 결과를 남기는 것이다. 빈칸에는 파일명, 컬럼명, 키 이름, 반복 제어 중 핵심 위치만 남겼기 때문에 학생이 데이터 구조를 이해해야 완성할 수 있다. 자주 보이는 오답은 컬럼명을 추측하거나, 중복과 실패 상태를 처리하지 않고 바로 저장하는 방식이다.

---

## 문제 12 정답 — 통합 CSV 행 만들기

~~~python
report_rows = []
for row in ready_courses:
    files = [item for item in manifest if item['course'] == row['course']]
    report_rows.append({'course': row['course'], 'teacher': row['teacher'], 'students': row['students'], 'file_count': len(files), 'status': row['status']})
print(report_rows)
~~~

### 왜 이 코드가 정답인지

이 답안은 문제에서 요구한 입력 구조를 먼저 읽고, 필요한 컬럼이나 selector만 선택한 뒤 결과를 명시적인 변수에 저장한다. `통합 CSV 행 만들기` 단계에서 중요한 점은 출력 하나를 맞히는 것이 아니라 이후 문제에서 재사용할 수 있는 안정적인 중간 결과를 남기는 것이다. 빈칸에는 파일명, 컬럼명, 키 이름, 반복 제어 중 핵심 위치만 남겼기 때문에 학생이 데이터 구조를 이해해야 완성할 수 있다. 자주 보이는 오답은 컬럼명을 추측하거나, 중복과 실패 상태를 처리하지 않고 바로 저장하는 방식이다.

---

## 문제 13 정답 — CSV와 JSON 저장

~~~python
write_csv('lesson10_portal_report.csv', report_rows)
summary = {'notice_count': len(notices), 'course_count': len(course_rows), 'ready_course_count': len(report_rows), 'file_count': len(manifest)}
Path('lesson10_portal_report.json').write_text(json.dumps(summary, ensure_ascii=False, indent=2), encoding='utf-8')
print(summary)
~~~

### 왜 이 코드가 정답인지

이 답안은 문제에서 요구한 입력 구조를 먼저 읽고, 필요한 컬럼이나 selector만 선택한 뒤 결과를 명시적인 변수에 저장한다. `CSV와 JSON 저장` 단계에서 중요한 점은 출력 하나를 맞히는 것이 아니라 이후 문제에서 재사용할 수 있는 안정적인 중간 결과를 남기는 것이다. 빈칸에는 파일명, 컬럼명, 키 이름, 반복 제어 중 핵심 위치만 남겼기 때문에 학생이 데이터 구조를 이해해야 완성할 수 있다. 자주 보이는 오답은 컬럼명을 추측하거나, 중복과 실패 상태를 처리하지 않고 바로 저장하는 방식이다.

---

## 문제 14 정답 — SQLite 저장과 조회

~~~python
conn = sqlite3.connect('lesson10_portal.db')
conn.execute('drop table if exists report')
conn.execute('create table report(course text, teacher text, students integer, file_count integer, status text)')
conn.executemany('insert into report values(:course, :teacher, :students, :file_count, :status)', report_rows)
print(conn.execute('select course, file_count from report order by students desc').fetchall())
conn.close()
~~~

### 왜 이 코드가 정답인지

이 답안은 문제에서 요구한 입력 구조를 먼저 읽고, 필요한 컬럼이나 selector만 선택한 뒤 결과를 명시적인 변수에 저장한다. `SQLite 저장과 조회` 단계에서 중요한 점은 출력 하나를 맞히는 것이 아니라 이후 문제에서 재사용할 수 있는 안정적인 중간 결과를 남기는 것이다. 빈칸에는 파일명, 컬럼명, 키 이름, 반복 제어 중 핵심 위치만 남겼기 때문에 학생이 데이터 구조를 이해해야 완성할 수 있다. 자주 보이는 오답은 컬럼명을 추측하거나, 중복과 실패 상태를 처리하지 않고 바로 저장하는 방식이다.

---

## 문제 15 정답 — 운영 메모 3문장 작성

~~~python
memo = [
    f"공지 {len(notices)}건, 과정 {len(course_rows)}건을 확인했습니다.",
    f"리포트 대상 과정은 {len(report_rows)}건이며 자료 파일은 {len(manifest)}개입니다.",
    f"최근 실행 수는 {metrics.get('runs', 0)}회이고 오류 수는 {metrics.get('errors', 0)}회입니다.",
]
print('\n'.join(memo))
~~~

### 왜 이 코드가 정답인지

이 답안은 문제에서 요구한 입력 구조를 먼저 읽고, 필요한 컬럼이나 selector만 선택한 뒤 결과를 명시적인 변수에 저장한다. `운영 메모 3문장 작성` 단계에서 중요한 점은 출력 하나를 맞히는 것이 아니라 이후 문제에서 재사용할 수 있는 안정적인 중간 결과를 남기는 것이다. 빈칸에는 파일명, 컬럼명, 키 이름, 반복 제어 중 핵심 위치만 남겼기 때문에 학생이 데이터 구조를 이해해야 완성할 수 있다. 자주 보이는 오답은 컬럼명을 추측하거나, 중복과 실패 상태를 처리하지 않고 바로 저장하는 방식이다.

### 보강 설명 1

교사는 학생 답안을 볼 때 값이 우연히 맞았는지, 입력 검증과 저장 순서를 설명할 수 있는지를 함께 확인한다. 특히 자동화 코드는 실패했을 때의 로그가 없으면 운영 코드로 보기 어렵다.

### 보강 설명 2

교사는 학생 답안을 볼 때 값이 우연히 맞았는지, 입력 검증과 저장 순서를 설명할 수 있는지를 함께 확인한다. 특히 자동화 코드는 실패했을 때의 로그가 없으면 운영 코드로 보기 어렵다.

### 보강 설명 3

교사는 학생 답안을 볼 때 값이 우연히 맞았는지, 입력 검증과 저장 순서를 설명할 수 있는지를 함께 확인한다. 특히 자동화 코드는 실패했을 때의 로그가 없으면 운영 코드로 보기 어렵다.

### 보강 설명 4

교사는 학생 답안을 볼 때 값이 우연히 맞았는지, 입력 검증과 저장 순서를 설명할 수 있는지를 함께 확인한다. 특히 자동화 코드는 실패했을 때의 로그가 없으면 운영 코드로 보기 어렵다.

### 보강 설명 5

교사는 학생 답안을 볼 때 값이 우연히 맞았는지, 입력 검증과 저장 순서를 설명할 수 있는지를 함께 확인한다. 특히 자동화 코드는 실패했을 때의 로그가 없으면 운영 코드로 보기 어렵다.

### 보강 설명 6

교사는 학생 답안을 볼 때 값이 우연히 맞았는지, 입력 검증과 저장 순서를 설명할 수 있는지를 함께 확인한다. 특히 자동화 코드는 실패했을 때의 로그가 없으면 운영 코드로 보기 어렵다.

### 보강 설명 7

교사는 학생 답안을 볼 때 값이 우연히 맞았는지, 입력 검증과 저장 순서를 설명할 수 있는지를 함께 확인한다. 특히 자동화 코드는 실패했을 때의 로그가 없으면 운영 코드로 보기 어렵다.

### 보강 설명 8

교사는 학생 답안을 볼 때 값이 우연히 맞았는지, 입력 검증과 저장 순서를 설명할 수 있는지를 함께 확인한다. 특히 자동화 코드는 실패했을 때의 로그가 없으면 운영 코드로 보기 어렵다.

### 보강 설명 9

교사는 학생 답안을 볼 때 값이 우연히 맞았는지, 입력 검증과 저장 순서를 설명할 수 있는지를 함께 확인한다. 특히 자동화 코드는 실패했을 때의 로그가 없으면 운영 코드로 보기 어렵다.

### 보강 설명 10

교사는 학생 답안을 볼 때 값이 우연히 맞았는지, 입력 검증과 저장 순서를 설명할 수 있는지를 함께 확인한다. 특히 자동화 코드는 실패했을 때의 로그가 없으면 운영 코드로 보기 어렵다.

### 보강 설명 11

교사는 학생 답안을 볼 때 값이 우연히 맞았는지, 입력 검증과 저장 순서를 설명할 수 있는지를 함께 확인한다. 특히 자동화 코드는 실패했을 때의 로그가 없으면 운영 코드로 보기 어렵다.

### 보강 설명 12

교사는 학생 답안을 볼 때 값이 우연히 맞았는지, 입력 검증과 저장 순서를 설명할 수 있는지를 함께 확인한다. 특히 자동화 코드는 실패했을 때의 로그가 없으면 운영 코드로 보기 어렵다.

### 보강 설명 13

교사는 학생 답안을 볼 때 값이 우연히 맞았는지, 입력 검증과 저장 순서를 설명할 수 있는지를 함께 확인한다. 특히 자동화 코드는 실패했을 때의 로그가 없으면 운영 코드로 보기 어렵다.

### 보강 설명 14

교사는 학생 답안을 볼 때 값이 우연히 맞았는지, 입력 검증과 저장 순서를 설명할 수 있는지를 함께 확인한다. 특히 자동화 코드는 실패했을 때의 로그가 없으면 운영 코드로 보기 어렵다.

### 보강 설명 15

교사는 학생 답안을 볼 때 값이 우연히 맞았는지, 입력 검증과 저장 순서를 설명할 수 있는지를 함께 확인한다. 특히 자동화 코드는 실패했을 때의 로그가 없으면 운영 코드로 보기 어렵다.

### 보강 설명 16

교사는 학생 답안을 볼 때 값이 우연히 맞았는지, 입력 검증과 저장 순서를 설명할 수 있는지를 함께 확인한다. 특히 자동화 코드는 실패했을 때의 로그가 없으면 운영 코드로 보기 어렵다.

### 보강 설명 17

교사는 학생 답안을 볼 때 값이 우연히 맞았는지, 입력 검증과 저장 순서를 설명할 수 있는지를 함께 확인한다. 특히 자동화 코드는 실패했을 때의 로그가 없으면 운영 코드로 보기 어렵다.

### 보강 설명 18

교사는 학생 답안을 볼 때 값이 우연히 맞았는지, 입력 검증과 저장 순서를 설명할 수 있는지를 함께 확인한다. 특히 자동화 코드는 실패했을 때의 로그가 없으면 운영 코드로 보기 어렵다.

### 보강 설명 19

교사는 학생 답안을 볼 때 값이 우연히 맞았는지, 입력 검증과 저장 순서를 설명할 수 있는지를 함께 확인한다. 특히 자동화 코드는 실패했을 때의 로그가 없으면 운영 코드로 보기 어렵다.

### 보강 설명 20

교사는 학생 답안을 볼 때 값이 우연히 맞았는지, 입력 검증과 저장 순서를 설명할 수 있는지를 함께 확인한다. 특히 자동화 코드는 실패했을 때의 로그가 없으면 운영 코드로 보기 어렵다.

### 보강 설명 21

교사는 학생 답안을 볼 때 값이 우연히 맞았는지, 입력 검증과 저장 순서를 설명할 수 있는지를 함께 확인한다. 특히 자동화 코드는 실패했을 때의 로그가 없으면 운영 코드로 보기 어렵다.

### 보강 설명 22

교사는 학생 답안을 볼 때 값이 우연히 맞았는지, 입력 검증과 저장 순서를 설명할 수 있는지를 함께 확인한다. 특히 자동화 코드는 실패했을 때의 로그가 없으면 운영 코드로 보기 어렵다.

### 보강 설명 23

교사는 학생 답안을 볼 때 값이 우연히 맞았는지, 입력 검증과 저장 순서를 설명할 수 있는지를 함께 확인한다. 특히 자동화 코드는 실패했을 때의 로그가 없으면 운영 코드로 보기 어렵다.

### 보강 설명 24

교사는 학생 답안을 볼 때 값이 우연히 맞았는지, 입력 검증과 저장 순서를 설명할 수 있는지를 함께 확인한다. 특히 자동화 코드는 실패했을 때의 로그가 없으면 운영 코드로 보기 어렵다.

### 보강 설명 25

교사는 학생 답안을 볼 때 값이 우연히 맞았는지, 입력 검증과 저장 순서를 설명할 수 있는지를 함께 확인한다. 특히 자동화 코드는 실패했을 때의 로그가 없으면 운영 코드로 보기 어렵다.
