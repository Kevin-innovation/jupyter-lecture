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

[![Open in Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/Kevin-innovation/jupyter-lecture/blob/main/python-web-automation/lectures/10/%EB%A0%88%EC%8A%A8%2010%20%E2%80%94%20%ED%86%B5%ED%95%A9%20%ED%94%84%EB%A1%9C%EC%A0%9D%ED%8A%B8%3A%20%EC%9E%90%EB%8F%99%ED%99%94%20%EB%A6%AC%ED%8F%AC%ED%8A%B8%20%EB%A7%8C%EB%93%A4%EA%B8%B0.ipynb)

> 선생님용 강의 노트북이다. 코랩에서 확인하려면 우측 상단 또는 아래의 **Open in Colab** 버튼을 클릭한다.

> 🔒 교사·관리자 전용. 학생에게 배포 금지.

통합 프로젝트: 자동화 리포트 만들기 실습 문제의 모범 답안이다. 출력값만 보지 말고 입력 구조, 검증 기준, 저장 구조, 운영 메모를 함께 확인한다.

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

포털 시작점은 portal_index.html이고 제목은 h1에 있다. 이 값을 먼저 확인해야 이후 링크 수집이 올바른 파일에서 시작되었는지 판단할 수 있다. text_or_empty()를 쓰면 selector가 비었을 때도 문자열 처리 오류를 피할 수 있어 수업용 검증 흐름에 맞다.

**채점 포인트:** 학생 답안이 같은 변수 흐름을 유지하는지 확인한다. 출력값만 맞아도 이전 단계에서 만든 데이터와 연결되지 않으면 최종 프로젝트에서는 감점한다. 저장 문제는 파일 생성 여부와 행 수를 함께 확인한다.

---

## 문제 2 정답 — 목차 링크 수집

~~~python
links = []
for a in index.select('a[data-page]'):
    links.append({'label': a.get_text(' ', strip=True), 'href': a['href']})
print(links)
~~~

### 왜 이 코드가 정답인지

a[data-page]는 포털에서 처리해야 할 하위 페이지만 표시하는 selector다. 링크 텍스트를 label, 실제 경로를 href로 함께 저장해야 다음 단계에서 사람이 읽는 이름과 코드가 사용할 경로를 모두 유지할 수 있다.

**채점 포인트:** 학생 답안이 같은 변수 흐름을 유지하는지 확인한다. 출력값만 맞아도 이전 단계에서 만든 데이터와 연결되지 않으면 최종 프로젝트에서는 감점한다. 저장 문제는 파일 생성 여부와 행 수를 함께 확인한다.

---

## 문제 3 정답 — 상대 URL을 절대 URL로 바꾸기

~~~python
base = 'https://lesson.local/portal/'
absolute = [urljoin(base, item['href']) for item in links]
print(absolute)
~~~

### 왜 이 코드가 정답인지

포털의 href는 상대 경로이므로 다른 기준 URL에서 실행하면 깨질 수 있다. urljoin()은 기준 경로와 상대 경로를 안전하게 합쳐 절대 URL을 만든다. href 키를 사용해야 앞 문제에서 수집한 링크 구조와 연결된다.

**채점 포인트:** 학생 답안이 같은 변수 흐름을 유지하는지 확인한다. 출력값만 맞아도 이전 단계에서 만든 데이터와 연결되지 않으면 최종 프로젝트에서는 감점한다. 저장 문제는 파일 생성 여부와 행 수를 함께 확인한다.

---

## 문제 4 정답 — 공지 카드 추출

~~~python
notice = parse_html('portal_notice.html')
notice_cards = notice.select('.notice-card')
print(len(notice_cards))
~~~

### 왜 이 코드가 정답인지

공지 페이지의 반복 단위는 .notice-card다. 먼저 카드 개수를 출력하면 selector가 너무 넓거나 좁지 않은지 빠르게 확인할 수 있다. 제목을 뽑기 전에 반복 단위를 확인하는 순서가 운영형 자동화의 기본이다.

**채점 포인트:** 학생 답안이 같은 변수 흐름을 유지하는지 확인한다. 출력값만 맞아도 이전 단계에서 만든 데이터와 연결되지 않으면 최종 프로젝트에서는 감점한다. 저장 문제는 파일 생성 여부와 행 수를 함께 확인한다.

---

## 문제 5 정답 — 공지 제목과 레벨 정리

~~~python
notices = []
for card in notice_cards:
    notices.append({'title': text_or_empty(card.select_one('.title')), 'level': card.get('data-level'), 'date': card.get('data-date')})
print(notices[:3])
~~~

### 왜 이 코드가 정답인지

공지의 화면 제목은 .title, 운영 분류는 data-level, 날짜는 data-date에 있다. HTML 텍스트와 data 속성을 함께 읽어야 사람이 보는 공지명과 코드가 분류할 레벨을 동시에 보존할 수 있다.

**채점 포인트:** 학생 답안이 같은 변수 흐름을 유지하는지 확인한다. 출력값만 맞아도 이전 단계에서 만든 데이터와 연결되지 않으면 최종 프로젝트에서는 감점한다. 저장 문제는 파일 생성 여부와 행 수를 함께 확인한다.

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

과정 정보는 표 행 단위로 들어 있으므로 tbody tr을 반복하고 각 행의 td를 순서대로 읽는다. 학생 수는 “18명”처럼 단위가 섞이므로 clean_int()로 숫자화해야 필터링과 정렬에 사용할 수 있다.

**채점 포인트:** 학생 답안이 같은 변수 흐름을 유지하는지 확인한다. 출력값만 맞아도 이전 단계에서 만든 데이터와 연결되지 않으면 최종 프로젝트에서는 감점한다. 저장 문제는 파일 생성 여부와 행 수를 함께 확인한다.

---

## 문제 7 정답 — 다운로드 manifest 읽기

~~~python
manifest = load_csv('download_manifest.csv')
print(manifest[0]['filename'])
~~~

### 왜 이 코드가 정답인지

download_manifest.csv는 화면 자료와 저장 기준을 비교하기 위한 기준표다. 첫 행의 filename을 출력하면 CSV 헤더가 예상대로 읽혔는지 확인할 수 있고, 이후 중복 키와 파일 개수 계산에 같은 컬럼을 사용한다.

**채점 포인트:** 학생 답안이 같은 변수 흐름을 유지하는지 확인한다. 출력값만 맞아도 이전 단계에서 만든 데이터와 연결되지 않으면 최종 프로젝트에서는 감점한다. 저장 문제는 파일 생성 여부와 행 수를 함께 확인한다.

---

## 문제 8 정답 — 상태 페이지 metric 추출

~~~python
status_page = parse_html('portal_status.html')
metrics = {item.get('data-name'): clean_int(item.get_text(' ', strip=True)) for item in status_page.select('.metric')}
print(metrics)
~~~

### 왜 이 코드가 정답인지

상태 페이지는 .metric 요소마다 이름을 data-name에 담고 값은 텍스트로 보여준다. 딕셔너리로 만들면 runs, submissions, errors 같은 값을 이름으로 바로 꺼낼 수 있어 최종 운영 메모 작성에 적합하다.

**채점 포인트:** 학생 답안이 같은 변수 흐름을 유지하는지 확인한다. 출력값만 맞아도 이전 단계에서 만든 데이터와 연결되지 않으면 최종 프로젝트에서는 감점한다. 저장 문제는 파일 생성 여부와 행 수를 함께 확인한다.

---

## 문제 9 정답 — 품질 규칙 읽기

~~~python
rules = load_json('quality_rules.json')
print(rules['min_students_for_report'])
~~~

### 왜 이 코드가 정답인지

리포트 기준은 코드에 직접 쓰지 않고 quality_rules.json에서 읽는다. min_students_for_report를 출력하면 필터 조건이 외부 설정에서 들어왔는지 확인할 수 있고, 기준 변경 시 코드 수정 범위를 줄일 수 있다.

**채점 포인트:** 학생 답안이 같은 변수 흐름을 유지하는지 확인한다. 출력값만 맞아도 이전 단계에서 만든 데이터와 연결되지 않으면 최종 프로젝트에서는 감점한다. 저장 문제는 파일 생성 여부와 행 수를 함께 확인한다.

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

리포트 대상은 상태가 운영 가능하고 학생 수 기준을 넘는 과정이다. status_ok()는 ready, published, open 같은 허용 상태를 한 곳에서 판정하고, JSON에서 읽은 최소 학생 수를 함께 적용해 기준을 명확히 한다.

**채점 포인트:** 학생 답안이 같은 변수 흐름을 유지하는지 확인한다. 출력값만 맞아도 이전 단계에서 만든 데이터와 연결되지 않으면 최종 프로젝트에서는 감점한다. 저장 문제는 파일 생성 여부와 행 수를 함께 확인한다.

---

## 문제 11 정답 — 자료 파일 중복 키 만들기

~~~python
keys = [make_key(row['course'], row['filename']) for row in manifest]
print(len(keys), len(set(keys)))
~~~

### 왜 이 코드가 정답인지

중복 검증은 과정명과 파일명을 함께 묶어야 의미가 있다. 파일명만 보면 다른 과정의 같은 이름 자료가 충돌할 수 있고, 과정명만 보면 여러 자료를 구분할 수 없다. 전체 키 개수와 고유 키 개수를 비교하면 중복 여부를 빠르게 볼 수 있다.

**채점 포인트:** 학생 답안이 같은 변수 흐름을 유지하는지 확인한다. 출력값만 맞아도 이전 단계에서 만든 데이터와 연결되지 않으면 최종 프로젝트에서는 감점한다. 저장 문제는 파일 생성 여부와 행 수를 함께 확인한다.

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

최종 리포트 행은 과정 정보와 manifest의 자료 개수를 결합한다. ready 과정별로 같은 course 값을 가진 파일만 골라야 실제 리포트 대상 과정의 자료 수를 정확히 계산할 수 있다.

**채점 포인트:** 학생 답안이 같은 변수 흐름을 유지하는지 확인한다. 출력값만 맞아도 이전 단계에서 만든 데이터와 연결되지 않으면 최종 프로젝트에서는 감점한다. 저장 문제는 파일 생성 여부와 행 수를 함께 확인한다.

---

## 문제 13 정답 — CSV와 JSON 저장

~~~python
write_csv('lesson10_portal_report.csv', report_rows)
summary = {'notice_count': len(notices), 'course_count': len(course_rows), 'ready_course_count': len(report_rows), 'file_count': len(manifest)}
Path('lesson10_portal_report.json').write_text(json.dumps(summary, ensure_ascii=False, indent=2), encoding='utf-8')
print(summary)
~~~

### 왜 이 코드가 정답인지

CSV는 상세 행을 저장하고 JSON은 전체 요약 숫자를 저장한다. len(manifest)를 넣어 자료 파일 수를 명시하면 최종 메모와 저장 파일을 비교할 수 있다. ensure_ascii=False는 한글 과정명이 읽기 좋게 저장되도록 한다.

**채점 포인트:** 학생 답안이 같은 변수 흐름을 유지하는지 확인한다. 출력값만 맞아도 이전 단계에서 만든 데이터와 연결되지 않으면 최종 프로젝트에서는 감점한다. 저장 문제는 파일 생성 여부와 행 수를 함께 확인한다.

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

SQLite 저장은 같은 리포트를 쿼리로 다시 확인하는 연습이다. report_rows 딕셔너리의 키와 테이블 컬럼을 맞추면 executemany()로 여러 행을 한 번에 넣을 수 있고, 학생 수 기준 정렬 조회까지 검증할 수 있다.

**채점 포인트:** 학생 답안이 같은 변수 흐름을 유지하는지 확인한다. 출력값만 맞아도 이전 단계에서 만든 데이터와 연결되지 않으면 최종 프로젝트에서는 감점한다. 저장 문제는 파일 생성 여부와 행 수를 함께 확인한다.

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

운영 메모는 결과를 사람이 이해하게 만드는 마지막 산출물이다. 공지와 과정 개수, 리포트 대상과 자료 파일 개수, 실행과 오류 metric을 포함하면 코드 실행 결과를 다음 담당자가 바로 확인할 수 있다. join(memo)를 출력해야 세 문장이 깔끔하게 분리된다.

**채점 포인트:** 학생 답안이 같은 변수 흐름을 유지하는지 확인한다. 출력값만 맞아도 이전 단계에서 만든 데이터와 연결되지 않으면 최종 프로젝트에서는 감점한다. 저장 문제는 파일 생성 여부와 행 수를 함께 확인한다.

## 교사용 점검 루틴

1. 학생이 portal_index.html에서 시작해 하위 페이지로 이동하는 흐름을 설명하는지 확인한다.
2. notice_cards, course_rows, manifest, metrics, rules가 각각 어떤 입력에서 만들어졌는지 묻는다.
3. CSV와 JSON 저장 뒤 파일이 실제로 생성되었는지 확인한다.
4. 운영 메모 3문장에 숫자가 들어 있는지 확인한다. 숫자가 없는 요약은 자동화 리포트로 보기 어렵다.


## 문항별 오답 진단표

| 문항 | 자주 나오는 오답 | 확인 질문 |
|---|---|---|
| 1 | 다른 HTML 파일을 열거나 h1 대신 body 전체를 출력함 | 시작 페이지가 무엇인지 설명할 수 있는가? |
| 2 | 모든 a 태그를 읽어 불필요한 링크까지 포함함 | data-page 속성이 왜 필요한가? |
| 3 | 문자열 더하기로 URL을 만들어 경로가 깨짐 | urljoin을 쓰면 어떤 상황에서 안전한가? |
| 4 | .notice-card 대신 .title을 반복 단위로 잡음 | 반복 단위와 추출 필드는 어떻게 다른가? |
| 5 | data-level을 텍스트에서 찾으려 함 | data 속성은 어떻게 읽는가? |
| 6 | 학생 수를 문자열 그대로 저장함 | 숫자 비교를 하려면 어떤 변환이 필요한가? |
| 7 | CSV를 문자열로만 읽고 DictReader를 쓰지 않음 | 헤더 이름으로 값을 꺼낼 수 있는가? |
| 8 | metric 이름 없이 값만 리스트로 저장함 | runs와 errors를 어떻게 구분할 것인가? |
| 9 | 기준 숫자를 코드에 직접 입력함 | 기준이 바뀌면 어디를 고칠 것인가? |
| 10 | 상태 조건이나 학생 수 조건 중 하나만 적용함 | 리포트 대상 기준 두 가지가 모두 들어갔는가? |
| 11 | 파일명만 key로 사용함 | 과정이 다르면 같은 파일명이 있어도 같은 자료인가? |
| 12 | ready_courses가 아닌 전체 course_rows를 저장함 | 필터링된 대상만 리포트에 들어갔는가? |
| 13 | CSV 또는 JSON 중 하나만 만들고 확인 출력을 생략함 | 저장 파일이 실제 생성됐는지 어떻게 확인하는가? |
| 14 | DB 연결을 닫지 않거나 컬럼 순서를 틀림 | 테이블 컬럼과 딕셔너리 키가 맞는가? |
| 15 | 숫자 없이 일반 감상문만 출력함 | 다음 담당자가 결과를 바로 이해할 수 있는가? |

## 채점 세부 기준

이 레슨은 통합 프로젝트이므로 문제 하나의 출력만 맞아도 전체 흐름이 끊기면 감점한다. 1~5번은 HTML 구조를 읽는 기초 확인, 6~10번은 표와 설정 파일을 결합하는 처리 확인, 11~15번은 저장과 보고 확인이다. 12문제 이상 통과가 기본 완료 기준이지만, 13번 또는 15번이 비어 있으면 최종 프로젝트 완성으로 보기 어렵다. 저장과 운영 메모가 없으면 실제 업무 자동화로 이어지지 않기 때문이다.

### 입력 구조 이해

학생이 selector를 외워서 채웠는지, fixture 구조를 읽고 채웠는지 구분해야 한다. 같은 결과가 나와도 HTML에서 card와 table row를 구분해 설명하지 못하면 다음 사이트 구조에서 바로 막힌다. 채점 중에는 “이 selector가 몇 개를 반환하나?”를 묻고, 학생이 len()으로 확인하게 한다.

### 변환 기준 이해

학생 수, metric, 파일 크기처럼 숫자로 다룰 값은 문자열에서 숫자로 바꿔야 한다. clean_int()를 사용하지 않고 문자열 비교를 하면 “9명”과 “18명”의 비교가 잘못될 수 있다. 이 문제를 설명할 수 있으면 자동화에서 데이터 타입이 왜 중요한지 이해한 것이다.

### 검증 기준 이해

ready_courses는 status_ok()와 min_students_for_report를 함께 통과한 과정이다. 조건 하나만 쓰면 리포트 대상이 너무 넓거나 좁아진다. 특히 draft 상태의 과정을 포함하는 답안은 운영 기준을 놓친 것이므로 다시 quality_rules와 status_ok()의 역할을 묻게 한다.

### 저장 구조 이해

CSV는 상세 목록, JSON은 요약, SQLite는 조회용이라는 차이를 설명할 수 있어야 한다. 학생이 모든 결과를 print만 하고 끝냈다면 운영 자동화가 아니라 실습 출력에 머문 것이다. 저장 파일이 만들어졌고, 그 파일에 어떤 행과 키가 들어 있는지까지 확인해야 한다.

### 운영 메모 이해

운영 메모는 “성공했습니다”가 아니라 “무엇을 몇 건 확인했고 무엇을 조심해야 하는지”를 말해야 한다. 공지 수, 과정 수, 대상 과정 수, 자료 파일 수, 실행 수, 오류 수 같은 숫자가 들어가면 다음 담당자가 상황을 빠르게 파악할 수 있다. 숫자가 하나도 없는 메모는 다시 작성하게 한다.

## 선생님 피드백 예시

- “selector는 맞지만 반복 단위가 title이라 이후 data-level을 함께 읽기 어렵다. 카드 전체를 반복 단위로 잡아보자.”
- “학생 수를 문자열로 두면 기준 비교가 흔들린다. clean_int()로 변환한 뒤 필터 조건에 넣자.”
- “CSV 저장은 됐지만 JSON 요약 숫자가 없다. 운영자가 빠르게 볼 수 있는 summary도 함께 남기자.”
- “운영 메모에 구체적인 숫자가 없어서 결과를 판단하기 어렵다. 공지 수, 대상 과정 수, 오류 수 중 최소 2개를 넣자.”
- “DB 저장까지 시도한 점은 좋다. 다만 close()가 빠지면 다음 실행에서 잠길 수 있으니 연결을 닫는 습관을 들이자.”


## 추가 검증 과제

수업 시간이 남으면 학생에게 아래 검증을 추가하게 한다. 이 과제는 정답 코드의 필수 조건은 아니지만, 통합 프로젝트의 완성도를 크게 높인다.

1. manifest 중복 여부를 boolean 값으로 summary에 넣는다.
2. draft 상태 과정이 report_rows에 들어가지 않았는지 assert로 확인한다.
3. JSON 저장 뒤 다시 읽어 ready_course_count가 report_rows 길이와 같은지 비교한다.
4. SQLite 조회 결과의 행 수가 CSV 행 수와 같은지 확인한다.
5. 운영 메모에 errors metric이 0보다 클 때만 “오류 확인 필요” 문장을 추가한다.

이 추가 검증은 학생이 단순 출력에서 운영 코드로 넘어가도록 돕는다. 자동화는 한 번 맞는 결과를 내는 것보다, 다음 실행에서 틀어진 부분을 빠르게 찾을 수 있어야 한다. 특히 최종 프로젝트에서는 저장 파일과 요약 숫자가 서로 맞는지 확인하는 습관을 강조한다.

## 루브릭

- 5점: fixture만 사용하고 외부 요청이 없다.
- 5점: HTML, CSV, JSON 입력을 모두 읽었다.
- 5점: 상태와 학생 수 기준으로 리포트 대상을 필터링했다.
- 5점: manifest 기준으로 자료 개수를 계산하거나 중복 키를 확인했다.
- 5점: CSV 또는 JSON 저장 파일을 생성했다.
- 5점: 저장 뒤 행 수 또는 요약 숫자를 출력했다.
- 5점: 운영 메모 3문장에 구체적인 숫자가 들어 있다.
- 5점: 코드가 위에서 아래로 재실행 가능하다.

총점보다 중요한 것은 흐름이다. 학생이 한 부분을 다르게 구현했더라도 입력, 검증, 저장, 보고가 연결되어 있으면 인정한다. 반대로 결과 숫자를 직접 적어 넣은 답안은 실행 가능성이 없으므로 점수를 낮게 준다.

## 빠른 재검수 체크

선생님은 학생 제출 노트북을 볼 때 전체 코드를 처음부터 읽기보다 마지막 출력부터 확인한다. 운영 메모에 공지 수, 과정 수, 리포트 대상 수, 오류 수가 들어 있으면 앞 단계가 대부분 연결된 것이다. 그 다음 저장 파일 이름을 보고, 마지막으로 ready_courses와 report_rows를 확인한다. 이 순서로 보면 피드백 시간이 줄어든다.

학생이 “코드는 실행됐는데 파일이 안 보여요”라고 말하면 현재 작업 폴더와 파일명을 먼저 출력하게 한다. 코랩에서는 파일이 노트북과 같은 위치에 있는 것이 아니라 런타임 작업 폴더에 생긴다. Path.cwd(), Path('lesson10_portal_report.csv').exists()를 확인하게 하면 저장 문제인지 경로 착각인지 구분할 수 있다.
