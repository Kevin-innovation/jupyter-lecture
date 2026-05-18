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

# 레슨 03 — 실습 문제 정답지

[![Open in Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/Kevin-innovation/jupyter-lecture/blob/main/python-web-automation/lectures/03/%EB%A0%88%EC%8A%A8%2003%20%E2%80%94%20HTML%20%ED%85%8C%EC%9D%B4%EB%B8%94%EA%B3%BC%20%EB%A6%AC%EC%8A%A4%ED%8A%B8%20%EB%8D%B0%EC%9D%B4%ED%84%B0%20%EC%A0%95%EB%A6%AC.ipynb)

HTML 테이블과 리스트 데이터 정리 실습 문제의 모범 답안이다. 출력만 맞는지보다 selector 기준, 반복 단위, 타입 변환, 저장 산출물을 함께 확인한다.

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

## 문제 1 정답 — 대시보드 제목 읽기

~~~python
dashboard_html = load_text('class_dashboard.html')
soup = BeautifulSoup(dashboard_html, 'html.parser')
print(soup.select_one('h1').text.strip())
~~~

### 왜 이 코드가 정답인지

load_text는 코랩과 로컬 경로 차이를 처리한다. BeautifulSoup으로 파싱한 뒤 select_one으로 h1 하나를 읽으면 페이지가 정상적으로 로드되었는지 빠르게 확인할 수 있다.

### 채점 포인트

- 파일을 읽은 뒤 selector가 실제 fixture 구조와 대응하는지 확인한다.
- 출력 형태가 문제 요구와 맞는지 확인한다.
- 이전 셀 변수를 재사용하는 문제는 실행 순서도 함께 확인한다.

### 자주 보이는 오답

- id selector를 빼고 th 전체를 읽어 다른 표와 섞는 경우가 있다.
- selector가 너무 넓어 불필요한 태그가 함께 잡힌다.
- 결과는 비슷하지만 저장 파일이나 요약 key가 문제 요구와 다르다.

---

## 문제 2 정답 — 테이블 헤더 추출하기

~~~python
headers = [th.text.strip() for th in soup.select('#class-table thead th')]
print(headers)
~~~

### 왜 이 코드가 정답인지

thead th만 선택하면 본문 td와 섞이지 않는다. 헤더를 먼저 리스트로 만들면 이후 각 행을 dict(zip(headers, cells)) 형태로 안정적으로 변환할 수 있다.

### 채점 포인트

- 리스트 길이와 첫 값을 함께 출력해 누락을 빠르게 찾는다.
- 출력 형태가 문제 요구와 맞는지 확인한다.
- 이전 셀 변수를 재사용하는 문제는 실행 순서도 함께 확인한다.

### 자주 보이는 오답

- td 순서를 직접 인덱스로만 처리해 컬럼 변경에 약해지는 경우가 있다.
- selector가 너무 넓어 불필요한 태그가 함께 잡힌다.
- 결과는 비슷하지만 저장 파일이나 요약 key가 문제 요구와 다르다.

---

## 문제 3 정답 — 테이블 행 개수 세기

~~~python
rows = soup.select('#class-table tbody tr')
print('rows:', len(rows))
~~~

### 왜 이 코드가 정답인지

행 개수는 selector가 맞는지 확인하는 기본 검증이다. tbody tr을 선택하면 헤더 행을 제외한 실제 학생 데이터만 얻는다.

### 채점 포인트

- 문자열 숫자를 계산 전에 변환했는지 확인한다.
- 출력 형태가 문제 요구와 맞는지 확인한다.
- 이전 셀 변수를 재사용하는 문제는 실행 순서도 함께 확인한다.

### 자주 보이는 오답

- 퍼센트 문자열을 숫자로 바꾸지 않고 비교하는 경우가 있다.
- selector가 너무 넓어 불필요한 태그가 함께 잡힌다.
- 결과는 비슷하지만 저장 파일이나 요약 key가 문제 요구와 다르다.

---

## 문제 4 정답 — 첫 행 딕셔너리 만들기

~~~python
cells = [td.text.strip() for td in rows[0].select('td')]
record = dict(zip(headers, cells))
print(record)
~~~

### 왜 이 코드가 정답인지

td 값의 순서와 th 헤더의 순서가 같기 때문에 zip으로 묶을 수 있다. 인덱스로 컬럼명을 직접 적는 방식보다 컬럼 변경에 강하다.

### 채점 포인트

- 저장 산출물의 컬럼 이름과 행 수가 요구사항과 맞는지 확인한다.
- 출력 형태가 문제 요구와 맞는지 확인한다.
- 이전 셀 변수를 재사용하는 문제는 실행 순서도 함께 확인한다.

### 자주 보이는 오답

- 카드 텍스트만 읽고 data-priority 속성을 놓치는 경우가 있다.
- selector가 너무 넓어 불필요한 태그가 함께 잡힌다.
- 결과는 비슷하지만 저장 파일이나 요약 key가 문제 요구와 다르다.

---

## 문제 5 정답 — 전체 테이블 리스트 만들기

~~~python
records = []
for tr in rows:
    cells = [td.text.strip() for td in tr.select('td')]
    records.append(dict(zip(headers, cells)))
print(records[0])
print(len(records))
~~~

### 왜 이 코드가 정답인지

모든 행을 같은 형태로 바꾸면 필터링, 정렬, 저장 단계에서 같은 key를 사용할 수 있다. len(records)를 출력해 누락 여부도 함께 확인한다.

### 채점 포인트

- 실제 사이트 확장 시 요청 간격과 개인정보 여부를 설명하게 한다.
- 출력 형태가 문제 요구와 맞는지 확인한다.
- 이전 셀 변수를 재사용하는 문제는 실행 순서도 함께 확인한다.

### 자주 보이는 오답

- CSV에 header를 쓰지 않아 결과 파일 의미가 불분명해지는 경우가 있다.
- selector가 너무 넓어 불필요한 태그가 함께 잡힌다.
- 결과는 비슷하지만 저장 파일이나 요약 key가 문제 요구와 다르다.

---

## 문제 6 정답 — 진도율과 통과율 숫자 변환하기

~~~python
for row in records:
    row['progress_num'] = clean_int(row['progress'])
    row['passed_num'] = int(row['passed'])
    row['submissions_num'] = int(row['submissions'])
    row['pass_rate'] = round(row['passed_num'] / row['submissions_num'] * 100, 1)
print(records[0]['student'], records[0]['progress_num'], records[0]['pass_rate'])
~~~

### 왜 이 코드가 정답인지

progress는 퍼센트 기호가 붙은 문자열이고 passed, submissions도 문자열이다. 숫자로 변환해야 비교와 평균 계산이 정확해진다.

### 채점 포인트

- 파일을 읽은 뒤 selector가 실제 fixture 구조와 대응하는지 확인한다.
- 출력 형태가 문제 요구와 맞는지 확인한다.
- 이전 셀 변수를 재사용하는 문제는 실행 순서도 함께 확인한다.

### 자주 보이는 오답

- id selector를 빼고 th 전체를 읽어 다른 표와 섞는 경우가 있다.
- selector가 너무 넓어 불필요한 태그가 함께 잡힌다.
- 결과는 비슷하지만 저장 파일이나 요약 key가 문제 요구와 다르다.

---

## 문제 7 정답 — 완료 기준 학생 필터링

~~~python
ready_students = [row['student'] for row in records if row['status'] == 'active' and row['progress_num'] >= 80]
print(ready_students)
~~~

### 왜 이 코드가 정답인지

진도율만 높아도 watch 상태라면 운영상 따로 확인해야 한다. 상태와 숫자 기준을 함께 쓰면 실제 수업 판단에 가까운 필터가 된다.

### 채점 포인트

- 리스트 길이와 첫 값을 함께 출력해 누락을 빠르게 찾는다.
- 출력 형태가 문제 요구와 맞는지 확인한다.
- 이전 셀 변수를 재사용하는 문제는 실행 순서도 함께 확인한다.

### 자주 보이는 오답

- td 순서를 직접 인덱스로만 처리해 컬럼 변경에 약해지는 경우가 있다.
- selector가 너무 넓어 불필요한 태그가 함께 잡힌다.
- 결과는 비슷하지만 저장 파일이나 요약 key가 문제 요구와 다르다.

---

## 문제 8 정답 — 코스별 평균 진도 계산

~~~python
summary = {}
for row in records:
    key = row['course']
    summary.setdefault(key, {'total': 0, 'count': 0})
    summary[key]['total'] += row['progress_num']
    summary[key]['count'] += 1
averages = {k: round(v['total'] / v['count'], 1) for k, v in summary.items()}
print(averages)
~~~

### 왜 이 코드가 정답인지

코스별 평균을 만들려면 같은 course에 속한 학생의 progress_num을 누적해야 한다. total과 count를 분리하면 평균 계산 과정이 분명해진다.

### 채점 포인트

- 문자열 숫자를 계산 전에 변환했는지 확인한다.
- 출력 형태가 문제 요구와 맞는지 확인한다.
- 이전 셀 변수를 재사용하는 문제는 실행 순서도 함께 확인한다.

### 자주 보이는 오답

- 퍼센트 문자열을 숫자로 바꾸지 않고 비교하는 경우가 있다.
- selector가 너무 넓어 불필요한 태그가 함께 잡힌다.
- 결과는 비슷하지만 저장 파일이나 요약 key가 문제 요구와 다르다.

---

## 문제 9 정답 — 코스별 학생 수 세기

~~~python
course_counts = Counter(row['course'] for row in records)
print(dict(course_counts))
~~~

### 왜 이 코드가 정답인지

개수 세기는 Counter가 가장 간단하다. 코스별 학생 수는 평균과 함께 운영 화면에서 자주 필요한 요약 값이다.

### 채점 포인트

- 저장 산출물의 컬럼 이름과 행 수가 요구사항과 맞는지 확인한다.
- 출력 형태가 문제 요구와 맞는지 확인한다.
- 이전 셀 변수를 재사용하는 문제는 실행 순서도 함께 확인한다.

### 자주 보이는 오답

- 카드 텍스트만 읽고 data-priority 속성을 놓치는 경우가 있다.
- selector가 너무 넓어 불필요한 태그가 함께 잡힌다.
- 결과는 비슷하지만 저장 파일이나 요약 key가 문제 요구와 다르다.

---

## 문제 10 정답 — 피드백 카드 읽기

~~~python
feedback_soup = BeautifulSoup(load_text('feedback_cards.html'), 'html.parser')
cards = feedback_soup.select('article.feedback-card')
print(len(cards))
print(cards[0].select_one('.title').text.strip())
~~~

### 왜 이 코드가 정답인지

카드형 UI는 표 헤더가 없으므로 카드 하나를 반복 단위로 잡아야 한다. article.feedback-card를 선택하면 제목, 요약, priority 속성을 한 번에 다룰 수 있다.

### 채점 포인트

- 실제 사이트 확장 시 요청 간격과 개인정보 여부를 설명하게 한다.
- 출력 형태가 문제 요구와 맞는지 확인한다.
- 이전 셀 변수를 재사용하는 문제는 실행 순서도 함께 확인한다.

### 자주 보이는 오답

- CSV에 header를 쓰지 않아 결과 파일 의미가 불분명해지는 경우가 있다.
- selector가 너무 넓어 불필요한 태그가 함께 잡힌다.
- 결과는 비슷하지만 저장 파일이나 요약 key가 문제 요구와 다르다.

---

## 문제 11 정답 — 긴급 피드백만 필터링

~~~python
urgent = []
for card in cards:
    if card['data-priority'] == 'high':
        urgent.append(card.select_one('.title').text.strip())
print(urgent)
~~~

### 왜 이 코드가 정답인지

data-priority는 사람이 보는 문장보다 자동화 기준으로 쓰기 좋다. high 값만 필터링하면 운영자가 먼저 확인해야 할 카드를 빠르게 분리할 수 있다.

### 채점 포인트

- 파일을 읽은 뒤 selector가 실제 fixture 구조와 대응하는지 확인한다.
- 출력 형태가 문제 요구와 맞는지 확인한다.
- 이전 셀 변수를 재사용하는 문제는 실행 순서도 함께 확인한다.

### 자주 보이는 오답

- id selector를 빼고 th 전체를 읽어 다른 표와 섞는 경우가 있다.
- selector가 너무 넓어 불필요한 태그가 함께 잡힌다.
- 결과는 비슷하지만 저장 파일이나 요약 key가 문제 요구와 다르다.

---

## 문제 12 정답 — todo 상태 세기

~~~python
todo_soup = BeautifulSoup(load_text('todo_list.html'), 'html.parser')
counts = {}
for item in todo_soup.select('li.todo-item'):
    status = item['data-status']
    counts[status] = counts.get(status, 0) + 1
print(counts)
~~~

### 왜 이 코드가 정답인지

리스트 텍스트에는 할 일 이름만 있고 상태는 속성에 들어 있다. data-status를 읽어야 진행 상태별 요약을 만들 수 있다.

### 채점 포인트

- 리스트 길이와 첫 값을 함께 출력해 누락을 빠르게 찾는다.
- 출력 형태가 문제 요구와 맞는지 확인한다.
- 이전 셀 변수를 재사용하는 문제는 실행 순서도 함께 확인한다.

### 자주 보이는 오답

- td 순서를 직접 인덱스로만 처리해 컬럼 변경에 약해지는 경우가 있다.
- selector가 너무 넓어 불필요한 태그가 함께 잡힌다.
- 결과는 비슷하지만 저장 파일이나 요약 key가 문제 요구와 다르다.

---

## 문제 13 정답 — 루브릭 CSV 점수 합계 계산

~~~python
rubrics = list(csv.DictReader(load_text('rubric.csv').splitlines()))
total_score = sum(int(row['max_score']) for row in rubrics)
print([row['criterion'] for row in rubrics])
print(total_score)
~~~

### 왜 이 코드가 정답인지

CSV에서 읽은 숫자는 문자열이다. int 변환 후 합산해야 총점이 정확하고, criterion을 출력하면 어떤 기준을 읽었는지도 검증할 수 있다.

### 채점 포인트

- 문자열 숫자를 계산 전에 변환했는지 확인한다.
- 출력 형태가 문제 요구와 맞는지 확인한다.
- 이전 셀 변수를 재사용하는 문제는 실행 순서도 함께 확인한다.

### 자주 보이는 오답

- 퍼센트 문자열을 숫자로 바꾸지 않고 비교하는 경우가 있다.
- selector가 너무 넓어 불필요한 태그가 함께 잡힌다.
- 결과는 비슷하지만 저장 파일이나 요약 key가 문제 요구와 다르다.

---

## 문제 14 정답 — 통합 운영 요약 만들기

~~~python
operation_summary = {
    'students': len(records),
    'courses': len(course_counts),
    'urgent_feedback': len(urgent),
    'pending_todos': counts.get('pending', 0),
}
print(operation_summary)
~~~

### 왜 이 코드가 정답인지

여러 출처에서 뽑은 값을 하나의 딕셔너리로 묶으면 이후 저장과 보고가 쉬워진다. get을 쓰면 pending이 없을 때도 KeyError 없이 0으로 처리할 수 있다.

### 채점 포인트

- 저장 산출물의 컬럼 이름과 행 수가 요구사항과 맞는지 확인한다.
- 출력 형태가 문제 요구와 맞는지 확인한다.
- 이전 셀 변수를 재사용하는 문제는 실행 순서도 함께 확인한다.

### 자주 보이는 오답

- 카드 텍스트만 읽고 data-priority 속성을 놓치는 경우가 있다.
- selector가 너무 넓어 불필요한 태그가 함께 잡힌다.
- 결과는 비슷하지만 저장 파일이나 요약 key가 문제 요구와 다르다.

---

## 문제 15 정답 — 코스 요약 CSV 저장하기

~~~python
with open('lesson03_course_summary.csv', 'w', newline='', encoding='utf-8') as f:
    writer = csv.DictWriter(f, fieldnames=['course', 'students', 'avg_progress'])
    writer.writeheader()
    for course, avg in sorted(averages.items()):
        writer.writerow({'course': course, 'students': course_counts[course], 'avg_progress': avg})
print('saved:', 'lesson03_course_summary.csv', len(averages))
~~~

### 왜 이 코드가 정답인지

CSV는 헤더가 있어야 사람이 열었을 때 의미를 바로 알 수 있다. averages와 course_counts를 같은 course key로 묶어 저장하면 요약 데이터가 재사용 가능한 산출물이 된다.

### 채점 포인트

- 실제 사이트 확장 시 요청 간격과 개인정보 여부를 설명하게 한다.
- 출력 형태가 문제 요구와 맞는지 확인한다.
- 이전 셀 변수를 재사용하는 문제는 실행 순서도 함께 확인한다.

### 자주 보이는 오답

- CSV에 header를 쓰지 않아 결과 파일 의미가 불분명해지는 경우가 있다.
- selector가 너무 넓어 불필요한 태그가 함께 잡힌다.
- 결과는 비슷하지만 저장 파일이나 요약 key가 문제 요구와 다르다.

---

## 교사용 마무리 점검

15문제 중 12문제 이상 맞으면 기본 통과로 본다. 다만 문제 5, 8, 12, 15는 이후 자동화 레슨의 기반이 되므로 반드시 피드백한다. 학생이 코드만 맞힌 경우에도 반복 단위, 타입 변환, 저장 산출물의 의미를 말로 설명하게 한다.

---

## 교사용 상세 피드백 기준

### table 단계

문제 1~5는 이후 모든 자동화의 기반이다. 학생이 soup.select를 썼는지보다 어떤 반복 단위를 잡았는지 말로 설명할 수 있는지 확인한다. #class-table thead th와 #class-table tbody tr의 차이를 설명하지 못하면 다음 카드형 UI에서도 selector를 무작정 복사할 가능성이 높다.

평가할 때는 headers 길이와 cells 길이가 같은지 함께 묻는다. 둘이 다르면 dict(zip(...))은 오류 없이 일부 값을 버릴 수 있다. 오류가 나지 않는다고 맞는 코드가 아니라는 점을 알려준다.

### 타입 변환 단계

문제 6~9에서는 progress, passed, submissions를 숫자로 바꾸는 이유를 반드시 확인한다. 학생이 출력 결과만 보고 넘어가면 문자열 비교와 숫자 비교 차이를 놓칠 수 있다. 예를 들어 '100%'와 '9%'를 문자열로 비교하면 기대와 다른 결과가 나온다. progress_num처럼 원본 필드를 보존하면서 새 숫자 필드를 만드는 방식을 권장한다.

Counter와 평균 계산은 정답 코드와 조금 달라도 인정할 수 있다. 단, 코스별 학생 수와 평균 진도 계산이 같은 원본 records에서 나온 값이어야 한다. 서로 다른 selector 결과를 섞으면 보고서 숫자가 맞지 않는다.

### card/list 단계

문제 10~12에서는 텍스트와 속성을 구분해야 한다. feedback 카드의 title은 화면 텍스트지만 priority는 data-priority 속성이다. todo 항목도 텍스트만 읽으면 상태 정보를 잃는다. 실제 UI 자동화에서는 화면에 보이지 않는 속성이 더 안정적인 기준이 되는 경우가 많다는 점을 설명한다.

학생이 soup.select('.title')처럼 전체 문서에서 바로 제목을 찾는 방식으로 풀 수도 있다. 다만 카드별 priority와 묶어야 하는 문제에서는 반드시 card 내부에서 select_one을 사용해야 한다. 데이터끼리 짝이 맞는지가 핵심이다.

### 저장 단계

문제 13~15는 운영 자동화의 끝부분이다. 저장 파일이 만들어지는 것뿐 아니라 헤더, 행 수, 컬럼 의미가 맞는지 확인한다. 학생이 writerow를 반복문 밖에 두면 마지막 값만 저장되거나 빈 파일이 만들어질 수 있다. 저장 후에는 파일명을 출력하고 행 수를 함께 출력하게 한다.

CSV는 사람이 열어 확인하기 좋고, JSON은 중첩 요약을 담기 좋다. 이번 문제는 CSV 저장을 요구하지만 최종 미션에서는 JSON을 추가로 저장해도 좋다. 단, 산출물 이름과 내용이 제출 요건과 맞아야 한다.

## 문제별 빠른 확인표

| 문제 | 핵심 검수 | 통과 기준 |
|---:|---|---|
| 1 | 파일 로드와 h1 selector | 제목 한 줄 출력 |
| 2 | thead th 선택 | 헤더 리스트 출력 |
| 3 | tbody tr 선택 | 원본 행 개수 출력 |
| 4 | headers와 cells 매핑 | 첫 행 dict 출력 |
| 5 | 모든 행 반복 | records 길이 확인 |
| 6 | 숫자 필드 추가 | progress_num과 pass_rate 생성 |
| 7 | 복합 조건 필터 | active와 진도 기준 동시 적용 |
| 8 | course별 평균 | total/count 구조 사용 |
| 9 | Counter 사용 | course별 개수 출력 |
| 10 | 카드 반복 단위 | article.feedback-card 선택 |
| 11 | 속성 필터 | data-priority high만 추출 |
| 12 | 리스트 상태 | data-status별 개수 계산 |
| 13 | CSV 숫자 합계 | max_score 합계 100 |
| 14 | 통합 요약 | 여러 출처 값 결합 |
| 15 | CSV 저장 | header와 행 수 출력 |

## 수업 중 멈춤 기준

학생 다수가 문제 4에서 막히면 table 구조 설명으로 돌아간다. 문제 6에서 막히면 타입 변환만 별도 예제로 보여준다. 문제 11에서 막히면 카드 하나를 출력해 title과 data-priority가 같은 article 안에 있음을 확인한다. 문제 15에서 막히면 CSV 저장 형식보다 먼저 저장하려는 rows의 형태를 print로 확인한다.

## 채점 운영 팁

정답 코드는 기준 예시일 뿐이다. 학생 코드가 다른 selector를 사용해도 같은 반복 단위와 같은 결과를 만들면 통과로 볼 수 있다. 예를 들어 '#class-table tbody tr' 대신 'table#class-table tbody tr'을 써도 의미는 같다. 반대로 결과가 우연히 한 번 맞더라도 반복 단위가 너무 넓거나 타입 변환이 빠졌다면 보완을 요구한다.

채점할 때는 다음 순서로 본다.

1. 실행 순서: 환경 셀부터 마지막 셀까지 새 런타임에서 동작하는가.
2. 데이터 개수: 원본 행, 카드, 리스트 항목 개수와 결과 개수가 일치하는가.
3. 데이터 형태: 리스트 안에 같은 key를 가진 딕셔너리가 들어 있는가.
4. 숫자 처리: 평균과 비교에 쓰는 값이 숫자인가.
5. 산출물: CSV 또는 JSON이 비어 있지 않고 헤더가 있는가.

학생이 막혔을 때 바로 답을 알려주기보다 다음 질문을 던진다. 지금 선택한 selector는 몇 개를 반환하는가. 첫 번째 item 안에는 어떤 하위 태그가 있는가. 지금 비교하는 값의 type은 무엇인가. 저장하려는 row를 print하면 어떤 딕셔너리인가. 이 네 질문으로 대부분의 문제를 해결할 수 있다.

## 확장 답안 허용 기준

우수 학생은 함수를 더 많이 분리하거나 JSON 저장을 추가할 수 있다. parse_table_records, summarize_courses, parse_feedback_cards처럼 이름이 분명한 함수로 나누면 가산점을 줄 수 있다. 단, 함수가 많아져도 최종 산출물은 요구한 CSV 또는 JSON 파일로 남아야 한다. 화면 출력만 있고 저장 파일이 없으면 운영 자동화 완성으로 보지 않는다.
## 재실행 검수 기준

교사용 검수에서는 노트북을 새 런타임에서 실행했을 때도 같은 파일이 생성되는지 확인한다. 중간 셀에만 존재하는 임시 변수에 기대어 동작하는 답안은 실제 수업 운영 자동화로 보기 어렵다. 저장 파일을 삭제한 뒤 다시 실행해도 같은 이름과 같은 행 수가 나오면 안정성이 높다.

