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

# 레슨 03 — 실습 문제

[![Open in Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/Kevin-innovation/jupyter-lecture/blob/main/python-web-automation/lectures/03/%5B%ED%95%99%EC%83%9D%EC%9A%A9%5D%20%EB%A0%88%EC%8A%A8%2003%20%E2%80%94%20HTML%20%ED%85%8C%EC%9D%B4%EB%B8%94%EA%B3%BC%20%EB%A6%AC%EC%8A%A4%ED%8A%B8%20%EB%8D%B0%EC%9D%B4%ED%84%B0%20%EC%A0%95%EB%A6%AC.ipynb)

HTML 테이블과 리스트 데이터 정리 레슨의 학생용 문제 노트북이다. 강의 노트북을 먼저 실행한 뒤 빈칸을 직접 채운다. 문제는 table 구조 확인에서 시작해 카드, 리스트, CSV 요약 저장까지 이어진다.

## 통과 기준

- 총 15문제 중 12문제 이상 정상 출력이면 통과.
- 문제 1~5는 table 구조와 행 변환, 6~9는 숫자 변환과 코스 요약, 10~15는 카드, 리스트, CSV 저장이다.
- 정답값과 완성 코드는 적지 않는다. fixture 구조, 출력 형태, 저장 파일 이름을 보고 직접 판단한다.

## 0. 환경 셀

~~~python
import os
import re
import csv
import json
from pathlib import Path
from collections import Counter, defaultdict

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
    DATA_BASE = 'https://raw.githubusercontent.com/Kevin-innovation/jupyter-lecture/main/python-web-automation/lectures/03/data'
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

def load_bytes(filename):
    if DATA_BASE.startswith('http'):
        url = f'{DATA_BASE}/{filename}'
        response = requests.get(url, timeout=10, headers={'User-Agent': 'D-Lab-Lesson/1.0'})
        response.raise_for_status()
        return response.content
    return Path(DATA_BASE, filename).read_bytes()

def read_soup(filename):
    return BeautifulSoup(load_text(filename), 'html.parser')

def clean_int(text):
    return int(re.sub(r'[^0-9]', '', str(text)))

def percent_to_int(text):
    return clean_int(text)

def safe_text(node, default=''):
    return node.text.strip() if node else default

print('colab:', IS_COLAB)
print('data base:', DATA_BASE)
~~~

---

## 문제 1 — 대시보드 제목 읽기

class_dashboard.html을 읽고 h1 제목을 출력한다. 파일 로드와 BeautifulSoup 변환을 한 번에 확인하는 문제다.

**기대 결과 형태**: 수업 운영 대시보드 제목이 한 줄로 출력된다.

**빈칸 힌트**: 파일은 load_text로 읽고, HTML 객체는 BeautifulSoup으로 만든다.

~~~python
dashboard_html = ____(____)
soup = ____(dashboard_html, 'html.parser')
print(soup.____('____').text.strip())
~~~

---

## 문제 2 — 테이블 헤더 추출하기

수업 운영 표의 th 텍스트를 읽어 딕셔너리 key로 사용할 헤더 목록을 만든다.

**기대 결과 형태**: student, course, level 등 컬럼명이 리스트로 출력된다.

**빈칸 힌트**: 헤더는 #class-table의 thead th에 들어 있다.

~~~python
headers = [th.text.strip() for th in soup.select('____')]
print(headers)
~~~

---

## 문제 3 — 테이블 행 개수 세기

tbody 안의 tr을 모두 선택하고 학생 행이 몇 개인지 확인한다.

**기대 결과 형태**: rows: 숫자 형태로 행 개수가 출력된다.

**빈칸 힌트**: 데이터 행은 #class-table tbody tr 구조다.

~~~python
rows = soup.select('____')
print('rows:', ____)
~~~

---

## 문제 4 — 첫 행 딕셔너리 만들기

첫 번째 tr의 td 텍스트를 headers와 묶어 한 학생 기록을 만든다.

**기대 결과 형태**: student, course, progress 등을 포함한 딕셔너리가 출력된다.

**빈칸 힌트**: 첫 행은 rows[0], 셀은 td로 선택한다.

~~~python
cells = [td.text.strip() for td in rows[____].select('____')]
record = dict(zip(____, ____))
print(record)
~~~

---

## 문제 5 — 전체 테이블 리스트 만들기

모든 학생 행을 같은 구조의 딕셔너리 리스트로 변환한다.

**기대 결과 형태**: 첫 레코드와 전체 레코드 수가 출력된다.

**빈칸 힌트**: 반복문 안에서 cells를 만들고 headers와 묶어 records에 추가한다.

~~~python
records = []
for tr in rows:
    cells = [td.text.strip() for td in tr.select('____')]
    records.append(dict(zip(____, ____)))
print(records[0])
print(len(records))
~~~

---

## 문제 6 — 진도율과 통과율 숫자 변환하기

progress 문자열과 passed/submissions 값을 계산 가능한 숫자로 바꾼다.

**기대 결과 형태**: 첫 학생 이름, 진도 숫자, 통과율이 출력된다.

**빈칸 힌트**: 퍼센트 문자는 clean_int로 제거하고, 통과율은 passed/submissions로 계산한다.

~~~python
for row in records:
    row['progress_num'] = ____(row['____'])
    row['passed_num'] = int(row['____'])
    row['submissions_num'] = int(row['____'])
    row['pass_rate'] = round(row['passed_num'] / row['submissions_num'] * 100, 1)
print(records[0]['student'], records[0]['____'], records[0]['____'])
~~~

---

## 문제 7 — 완료 기준 학생 필터링

active 상태이면서 진도율이 80 이상인 학생만 골라낸다.

**기대 결과 형태**: 조건을 만족하는 학생 이름 리스트가 출력된다.

**빈칸 힌트**: 조건은 status와 progress_num 두 가지를 동시에 확인한다.

~~~python
ready_students = [row['student'] for row in records if row['____'] == 'active' and row['____'] >= ____]
print(ready_students)
~~~

---

## 문제 8 — 코스별 평균 진도 계산

학생 records를 course별로 묶어 평균 진도율을 계산한다.

**기대 결과 형태**: 코스 이름과 평균 진도율 딕셔너리가 출력된다.

**빈칸 힌트**: summary에는 total과 count를 누적하고 마지막에 나누어 평균을 만든다.

~~~python
summary = {}
for row in records:
    key = row['____']
    summary.setdefault(key, {'total': 0, 'count': 0})
    summary[key]['total'] += row['____']
    summary[key]['count'] += 1
averages = {k: round(v['total'] / v['count'], 1) for k, v in summary.items()}
print(averages)
~~~

---

## 문제 9 — 코스별 학생 수 세기

Counter를 사용해 course 값이 몇 번 등장하는지 센다.

**기대 결과 형태**: Python, Web, C, Java별 학생 수가 출력된다.

**빈칸 힌트**: Counter에는 course 값만 뽑은 리스트를 넣는다.

~~~python
course_counts = ____(row['____'] for row in records)
print(dict(course_counts))
~~~

---

## 문제 10 — 피드백 카드 읽기

feedback_cards.html의 article.feedback-card를 읽고 첫 카드 제목을 확인한다.

**기대 결과 형태**: 카드 개수와 첫 카드 제목이 출력된다.

**빈칸 힌트**: 카드 반복 단위는 article.feedback-card이고 제목은 .title이다.

~~~python
feedback_soup = BeautifulSoup(load_text('____'), 'html.parser')
cards = feedback_soup.select('____')
print(len(cards))
print(cards[0].select_one('____').text.strip())
~~~

---

## 문제 11 — 긴급 피드백만 필터링

data-priority가 high인 카드 제목만 따로 모은다.

**기대 결과 형태**: 긴급 피드백 제목 리스트가 출력된다.

**빈칸 힌트**: priority는 화면 텍스트가 아니라 card의 data-priority 속성이다.

~~~python
urgent = []
for card in cards:
    if card['____'] == ____:
        urgent.append(card.select_one('____').text.strip())
print(urgent)
~~~

---

## 문제 12 — todo 상태 세기

todo_list.html의 li.todo-item을 읽어 pending, done, blocked 개수를 센다.

**기대 결과 형태**: 상태별 개수 딕셔너리가 출력된다.

**빈칸 힌트**: 상태는 data-status 속성에 있다.

~~~python
todo_soup = BeautifulSoup(load_text('____'), 'html.parser')
counts = {}
for item in todo_soup.select('____'):
    status = item['____']
    counts[status] = counts.get(status, 0) + 1
print(counts)
~~~

---

## 문제 13 — 루브릭 CSV 점수 합계 계산

rubric.csv를 읽고 max_score 합계가 얼마인지 계산한다.

**기대 결과 형태**: 평가 항목 이름과 총점이 출력된다.

**빈칸 힌트**: csv.DictReader는 문자열을 반환하므로 max_score를 int로 바꾼다.

~~~python
rubrics = list(csv.DictReader(load_text('____').splitlines()))
total_score = sum(int(row['____']) for row in rubrics)
print([row['____'] for row in rubrics])
print(total_score)
~~~

---

## 문제 14 — 통합 운영 요약 만들기

학생 행 수, 코스 수, 긴급 피드백 수, pending todo 수를 하나의 딕셔너리로 묶는다.

**기대 결과 형태**: 운영 요약 딕셔너리가 출력된다.

**빈칸 힌트**: records, course_counts, urgent, counts 변수를 재사용한다.

~~~python
operation_summary = {
    'students': ____,
    'courses': ____,
    'urgent_feedback': ____,
    'pending_todos': counts.get(____, 0),
}
print(operation_summary)
~~~

---

## 문제 15 — 코스 요약 CSV 저장하기

코스별 평균 진도와 학생 수를 lesson03_course_summary.csv로 저장한다.

**기대 결과 형태**: 저장된 파일명과 행 수가 출력된다.

**빈칸 힌트**: fieldnames는 저장할 컬럼 순서이고 writeheader 후 writerow를 반복한다.

~~~python
with open('lesson03_course_summary.csv', 'w', newline='', encoding='utf-8') as f:
    writer = csv.DictWriter(f, fieldnames=['course', 'students', 'avg_progress'])
    writer.____()
    for course, avg in sorted(averages.items()):
        writer.writerow({'course': course, 'students': course_counts[course], 'avg_progress': avg})
print('saved:', 'lesson03_course_summary.csv', ____)
~~~

---

## 제출 전 점검

- h1, th, tr, td selector가 fixture 구조와 맞는지 확인한다.
- progress, submissions, passed, max_score처럼 계산에 쓰는 값은 숫자로 변환한다.
- 저장 문제를 실행한 뒤 lesson03_course_summary.csv가 생성되는지 확인한다.
- 실제 사이트로 확장할 때는 요청 간격, 개인정보 여부, 이용 약관을 먼저 확인한다.

---

## 문제 풀이 기준

빈칸 문제는 정답 코드를 외우는 방식이 아니라 구조를 읽는 순서로 풀어야 한다. 문제를 풀 때는 아래 순서를 지킨다.

| 단계 | 확인할 것 | 예시 |
|---|---|---|
| 파일 확인 | 어떤 fixture를 읽는가 | class_dashboard.html |
| 반복 단위 | 무엇을 여러 번 선택하는가 | tbody tr, article.feedback-card, li.todo-item |
| 하위 값 | 텍스트와 속성 중 무엇을 읽는가 | .title 텍스트, data-priority 속성 |
| 타입 변환 | 숫자로 바꿀 값은 무엇인가 | progress, submissions, max_score |
| 출력 형태 | 리스트, 딕셔너리, CSV 중 무엇인가 | course별 평균 딕셔너리 |

문제 1~5는 테이블을 읽는 기본 흐름이다. 파일을 읽고 soup를 만든 뒤 headers와 rows를 분리한다. 문제 6~9는 records를 계산 가능한 형태로 바꾸는 단계다. 문제 10~12는 table이 아닌 카드, 리스트, CSV를 다룬다. 문제 13~15는 여러 결과를 하나로 합치고 저장한다.

## 학생 제출 전 자체 점검

- 빈칸을 채운 뒤 셀을 위에서 아래로 순서대로 실행했는가.
- selector가 빈 리스트를 반환하지 않는가.
- 숫자 비교에 쓰는 값이 int 또는 float로 변환되었는가.
- CSV 저장 문제에서 writeheader를 호출했는가.
- 생성한 CSV 파일 이름이 문제에서 요구한 이름과 같은가.
- 실제 사이트가 아니라 제공된 fixture만 사용했는가.

이 점검은 채점 전에 스스로 오류를 줄이기 위한 절차다. 특히 코랩에서는 이전 실행 결과가 남아 있을 수 있으므로 런타임을 다시 시작한 뒤 전체 실행해 보는 것이 좋다.

## 빈칸 난이도 안내

| 문제 | 난이도 | 막혔을 때 확인할 곳 |
|---:|---|---|
| 1~3 | 기본 | 파일명, BeautifulSoup 생성, table selector |
| 4~5 | 기본+ | headers와 cells 길이, td 선택 위치 |
| 6~8 | 중간 | clean_int, progress_num, course key |
| 9~12 | 중간 | Counter, article.feedback-card, data-status |
| 13~15 | 종합 | csv.DictReader, 통합 요약, DictWriter 저장 |

힌트의 빈칸은 정답을 거의 그대로 주지 않기 위해 함수 이름, selector, key 이름을 일부 비워 둔 것이다. 먼저 HTML 파일을 직접 열어 태그 이름과 속성 이름을 확인하고, 강의 노트북에서 같은 구조를 처리한 셀을 찾는다. 그런 다음 빈칸에 들어갈 값이 함수인지, 문자열 selector인지, 딕셔너리 key인지 구분해서 채운다.

## 오류가 났을 때 확인 순서

1. NameError가 나면 이전 셀을 실행했는지 확인한다.
2. NoneType 오류가 나면 selector 결과가 비었는지 확인한다.
3. KeyError가 나면 딕셔너리 key를 print(record.keys())로 확인한다.
4. TypeError가 나면 문자열 숫자를 int로 바꿨는지 확인한다.
5. 저장 파일이 비어 있으면 writerow가 반복문 안에 있는지 확인한다.

이 순서대로 보면 대부분의 오류는 스스로 좁힐 수 있다. 정답 코드를 바로 보는 것보다 오류 종류를 분류하는 능력이 이후 자동화 수업에서 더 중요하다.
## 마지막 실행 기준

제출 전에는 코드 셀을 부분 실행하지 말고 위에서 아래로 전체 실행한다. 이전 실행에서 남은 records나 counts 때문에 우연히 맞는 답처럼 보일 수 있기 때문이다. 전체 실행 후 마지막 셀에서 저장 파일명과 행 수가 출력되면 기본 흐름은 통과한 것으로 본다.

