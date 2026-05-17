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

> 🔒 교사·관리자 전용. 학생에게 배포 금지.

HTML 테이블과 리스트 데이터 정리 실습 문제의 모범 답안이다. 출력값만 보지 말고 selector, 타입 변환, 저장 흐름을 같이 확인한다.

## 0. 환경 셀

~~~python
import os
import re
import time
import csv
from pathlib import Path
from urllib.parse import urljoin, urlparse, parse_qs, urlencode

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

def clean_int(text):
    return int(re.sub(r'[^0-9]', '', str(text)))


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

이 답안은 문제에서 요구한 입력을 먼저 안정적인 자료구조로 바꾼 뒤 필요한 값만 선택한다. table, card, list마다 반복 단위가 다르므로 구조별 selector를 분리해야 한다. 중간 변수의 길이와 타입이 맞아야 뒤의 필터링, 저장, 요약이 모두 맞는다.

### 채점 포인트

- 입력 파일을 환경 셀의 헬퍼로 읽어 코랩과 로컬에서 모두 동작하는가.
- 반복 단위와 selector 또는 파일 경로가 fixture 구조와 정확히 대응하는가.
- 숫자, 날짜, 상대 경로처럼 후처리가 필요한 값을 그대로 두지 않았는가.
- 출력 형태가 문제 요구사항과 일치하는가.

### 자주 보이는 오답

- 이전 문제 selector를 그대로 복사해 현재 HTML 구조와 맞지 않는다.
- 문자열 숫자를 변환하지 않고 비교한다.
- 결과는 비슷하지만 중간 변수명이 불분명해 최종 미션에서 재사용하기 어렵다.

---

## 문제 2 정답 — 테이블 헤더 추출하기

~~~python
headers = [th.text.strip() for th in soup.select('#class-table thead th')]
print(headers)
~~~

### 왜 이 코드가 정답인지

이 답안은 문제에서 요구한 입력을 먼저 안정적인 자료구조로 바꾼 뒤 필요한 값만 선택한다. table, card, list마다 반복 단위가 다르므로 구조별 selector를 분리해야 한다. 중간 변수의 길이와 타입이 맞아야 뒤의 필터링, 저장, 요약이 모두 맞는다.

### 채점 포인트

- 입력 파일을 환경 셀의 헬퍼로 읽어 코랩과 로컬에서 모두 동작하는가.
- 반복 단위와 selector 또는 파일 경로가 fixture 구조와 정확히 대응하는가.
- 숫자, 날짜, 상대 경로처럼 후처리가 필요한 값을 그대로 두지 않았는가.
- 출력 형태가 문제 요구사항과 일치하는가.

### 자주 보이는 오답

- 이전 문제 selector를 그대로 복사해 현재 HTML 구조와 맞지 않는다.
- 문자열 숫자를 변환하지 않고 비교한다.
- 결과는 비슷하지만 중간 변수명이 불분명해 최종 미션에서 재사용하기 어렵다.

---

## 문제 3 정답 — 테이블 행 개수 세기

~~~python
rows = soup.select('#class-table tbody tr')
print('rows:', len(rows))
~~~

### 왜 이 코드가 정답인지

이 답안은 문제에서 요구한 입력을 먼저 안정적인 자료구조로 바꾼 뒤 필요한 값만 선택한다. table, card, list마다 반복 단위가 다르므로 구조별 selector를 분리해야 한다. 중간 변수의 길이와 타입이 맞아야 뒤의 필터링, 저장, 요약이 모두 맞는다.

### 채점 포인트

- 입력 파일을 환경 셀의 헬퍼로 읽어 코랩과 로컬에서 모두 동작하는가.
- 반복 단위와 selector 또는 파일 경로가 fixture 구조와 정확히 대응하는가.
- 숫자, 날짜, 상대 경로처럼 후처리가 필요한 값을 그대로 두지 않았는가.
- 출력 형태가 문제 요구사항과 일치하는가.

### 자주 보이는 오답

- 이전 문제 selector를 그대로 복사해 현재 HTML 구조와 맞지 않는다.
- 문자열 숫자를 변환하지 않고 비교한다.
- 결과는 비슷하지만 중간 변수명이 불분명해 최종 미션에서 재사용하기 어렵다.

---

## 문제 4 정답 — 첫 행 딕셔너리 만들기

~~~python
cells = [td.text.strip() for td in rows[0].select('td')]
record = dict(zip(headers, cells))
print(record)
~~~

### 왜 이 코드가 정답인지

이 답안은 문제에서 요구한 입력을 먼저 안정적인 자료구조로 바꾼 뒤 필요한 값만 선택한다. table, card, list마다 반복 단위가 다르므로 구조별 selector를 분리해야 한다. 중간 변수의 길이와 타입이 맞아야 뒤의 필터링, 저장, 요약이 모두 맞는다.

### 채점 포인트

- 입력 파일을 환경 셀의 헬퍼로 읽어 코랩과 로컬에서 모두 동작하는가.
- 반복 단위와 selector 또는 파일 경로가 fixture 구조와 정확히 대응하는가.
- 숫자, 날짜, 상대 경로처럼 후처리가 필요한 값을 그대로 두지 않았는가.
- 출력 형태가 문제 요구사항과 일치하는가.

### 자주 보이는 오답

- 이전 문제 selector를 그대로 복사해 현재 HTML 구조와 맞지 않는다.
- 문자열 숫자를 변환하지 않고 비교한다.
- 결과는 비슷하지만 중간 변수명이 불분명해 최종 미션에서 재사용하기 어렵다.

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

이 답안은 문제에서 요구한 입력을 먼저 안정적인 자료구조로 바꾼 뒤 필요한 값만 선택한다. table, card, list마다 반복 단위가 다르므로 구조별 selector를 분리해야 한다. 중간 변수의 길이와 타입이 맞아야 뒤의 필터링, 저장, 요약이 모두 맞는다.

### 채점 포인트

- 입력 파일을 환경 셀의 헬퍼로 읽어 코랩과 로컬에서 모두 동작하는가.
- 반복 단위와 selector 또는 파일 경로가 fixture 구조와 정확히 대응하는가.
- 숫자, 날짜, 상대 경로처럼 후처리가 필요한 값을 그대로 두지 않았는가.
- 출력 형태가 문제 요구사항과 일치하는가.

### 자주 보이는 오답

- 이전 문제 selector를 그대로 복사해 현재 HTML 구조와 맞지 않는다.
- 문자열 숫자를 변환하지 않고 비교한다.
- 결과는 비슷하지만 중간 변수명이 불분명해 최종 미션에서 재사용하기 어렵다.

---

## 문제 6 정답 — 진도율 숫자 변환하기

~~~python
for row in records:
    row['progress_num'] = clean_int(row['progress'])
print(records[0]['student'], records[0]['progress_num'])
~~~

### 왜 이 코드가 정답인지

이 답안은 문제에서 요구한 입력을 먼저 안정적인 자료구조로 바꾼 뒤 필요한 값만 선택한다. table, card, list마다 반복 단위가 다르므로 구조별 selector를 분리해야 한다. 중간 변수의 길이와 타입이 맞아야 뒤의 필터링, 저장, 요약이 모두 맞는다.

### 채점 포인트

- 입력 파일을 환경 셀의 헬퍼로 읽어 코랩과 로컬에서 모두 동작하는가.
- 반복 단위와 selector 또는 파일 경로가 fixture 구조와 정확히 대응하는가.
- 숫자, 날짜, 상대 경로처럼 후처리가 필요한 값을 그대로 두지 않았는가.
- 출력 형태가 문제 요구사항과 일치하는가.

### 자주 보이는 오답

- 이전 문제 selector를 그대로 복사해 현재 HTML 구조와 맞지 않는다.
- 문자열 숫자를 변환하지 않고 비교한다.
- 결과는 비슷하지만 중간 변수명이 불분명해 최종 미션에서 재사용하기 어렵다.

---

## 문제 7 정답 — 완료 기준 학생 필터링

~~~python
done_students = [row['student'] for row in records if row['progress_num'] >= 80]
print(done_students)
~~~

### 왜 이 코드가 정답인지

이 답안은 문제에서 요구한 입력을 먼저 안정적인 자료구조로 바꾼 뒤 필요한 값만 선택한다. table, card, list마다 반복 단위가 다르므로 구조별 selector를 분리해야 한다. 중간 변수의 길이와 타입이 맞아야 뒤의 필터링, 저장, 요약이 모두 맞는다.

### 채점 포인트

- 입력 파일을 환경 셀의 헬퍼로 읽어 코랩과 로컬에서 모두 동작하는가.
- 반복 단위와 selector 또는 파일 경로가 fixture 구조와 정확히 대응하는가.
- 숫자, 날짜, 상대 경로처럼 후처리가 필요한 값을 그대로 두지 않았는가.
- 출력 형태가 문제 요구사항과 일치하는가.

### 자주 보이는 오답

- 이전 문제 selector를 그대로 복사해 현재 HTML 구조와 맞지 않는다.
- 문자열 숫자를 변환하지 않고 비교한다.
- 결과는 비슷하지만 중간 변수명이 불분명해 최종 미션에서 재사용하기 어렵다.

---

## 문제 8 정답 — 코스별 평균 진도 계산

~~~python
summary = {}
for row in records:
    key = row['course']
    summary.setdefault(key, {'total': 0, 'count': 0})
    summary[key]['total'] += row['progress_num']
    summary[key]['count'] += 1
averages = {k: v['total'] / v['count'] for k, v in summary.items()}
print(averages)
~~~

### 왜 이 코드가 정답인지

이 답안은 문제에서 요구한 입력을 먼저 안정적인 자료구조로 바꾼 뒤 필요한 값만 선택한다. table, card, list마다 반복 단위가 다르므로 구조별 selector를 분리해야 한다. 중간 변수의 길이와 타입이 맞아야 뒤의 필터링, 저장, 요약이 모두 맞는다.

### 채점 포인트

- 입력 파일을 환경 셀의 헬퍼로 읽어 코랩과 로컬에서 모두 동작하는가.
- 반복 단위와 selector 또는 파일 경로가 fixture 구조와 정확히 대응하는가.
- 숫자, 날짜, 상대 경로처럼 후처리가 필요한 값을 그대로 두지 않았는가.
- 출력 형태가 문제 요구사항과 일치하는가.

### 자주 보이는 오답

- 이전 문제 selector를 그대로 복사해 현재 HTML 구조와 맞지 않는다.
- 문자열 숫자를 변환하지 않고 비교한다.
- 결과는 비슷하지만 중간 변수명이 불분명해 최종 미션에서 재사용하기 어렵다.

---

## 문제 9 정답 — 피드백 카드 읽기

~~~python
feedback_soup = BeautifulSoup(load_text('feedback_cards.html'), 'html.parser')
cards = feedback_soup.select('article.feedback-card')
print(len(cards))
print(cards[0].select_one('.title').text.strip())
~~~

### 왜 이 코드가 정답인지

이 답안은 문제에서 요구한 입력을 먼저 안정적인 자료구조로 바꾼 뒤 필요한 값만 선택한다. table, card, list마다 반복 단위가 다르므로 구조별 selector를 분리해야 한다. 중간 변수의 길이와 타입이 맞아야 뒤의 필터링, 저장, 요약이 모두 맞는다.

### 채점 포인트

- 입력 파일을 환경 셀의 헬퍼로 읽어 코랩과 로컬에서 모두 동작하는가.
- 반복 단위와 selector 또는 파일 경로가 fixture 구조와 정확히 대응하는가.
- 숫자, 날짜, 상대 경로처럼 후처리가 필요한 값을 그대로 두지 않았는가.
- 출력 형태가 문제 요구사항과 일치하는가.

### 자주 보이는 오답

- 이전 문제 selector를 그대로 복사해 현재 HTML 구조와 맞지 않는다.
- 문자열 숫자를 변환하지 않고 비교한다.
- 결과는 비슷하지만 중간 변수명이 불분명해 최종 미션에서 재사용하기 어렵다.

---

## 문제 10 정답 — 긴급 카드만 필터링

~~~python
urgent = []
for card in cards:
    if card['data-priority'] == 'high':
        urgent.append(card.select_one('.title').text.strip())
print(urgent)
~~~

### 왜 이 코드가 정답인지

이 답안은 문제에서 요구한 입력을 먼저 안정적인 자료구조로 바꾼 뒤 필요한 값만 선택한다. table, card, list마다 반복 단위가 다르므로 구조별 selector를 분리해야 한다. 중간 변수의 길이와 타입이 맞아야 뒤의 필터링, 저장, 요약이 모두 맞는다.

### 채점 포인트

- 입력 파일을 환경 셀의 헬퍼로 읽어 코랩과 로컬에서 모두 동작하는가.
- 반복 단위와 selector 또는 파일 경로가 fixture 구조와 정확히 대응하는가.
- 숫자, 날짜, 상대 경로처럼 후처리가 필요한 값을 그대로 두지 않았는가.
- 출력 형태가 문제 요구사항과 일치하는가.

### 자주 보이는 오답

- 이전 문제 selector를 그대로 복사해 현재 HTML 구조와 맞지 않는다.
- 문자열 숫자를 변환하지 않고 비교한다.
- 결과는 비슷하지만 중간 변수명이 불분명해 최종 미션에서 재사용하기 어렵다.

---

## 문제 11 정답 — todo 상태 세기

~~~python
todo_soup = BeautifulSoup(load_text('todo_list.html'), 'html.parser')
counts = {}
for item in todo_soup.select('li.todo-item'):
    status = item['data-status']
    counts[status] = counts.get(status, 0) + 1
print(counts)
~~~

### 왜 이 코드가 정답인지

이 답안은 문제에서 요구한 입력을 먼저 안정적인 자료구조로 바꾼 뒤 필요한 값만 선택한다. table, card, list마다 반복 단위가 다르므로 구조별 selector를 분리해야 한다. 중간 변수의 길이와 타입이 맞아야 뒤의 필터링, 저장, 요약이 모두 맞는다.

### 채점 포인트

- 입력 파일을 환경 셀의 헬퍼로 읽어 코랩과 로컬에서 모두 동작하는가.
- 반복 단위와 selector 또는 파일 경로가 fixture 구조와 정확히 대응하는가.
- 숫자, 날짜, 상대 경로처럼 후처리가 필요한 값을 그대로 두지 않았는가.
- 출력 형태가 문제 요구사항과 일치하는가.

### 자주 보이는 오답

- 이전 문제 selector를 그대로 복사해 현재 HTML 구조와 맞지 않는다.
- 문자열 숫자를 변환하지 않고 비교한다.
- 결과는 비슷하지만 중간 변수명이 불분명해 최종 미션에서 재사용하기 어렵다.

---

## 문제 12 정답 — 루브릭 CSV 읽기

~~~python
rubrics = list(csv.DictReader(load_text('rubric.csv').splitlines()))
for row in rubrics:
    print(row['criterion'], row['max_score'])
~~~

### 왜 이 코드가 정답인지

이 답안은 문제에서 요구한 입력을 먼저 안정적인 자료구조로 바꾼 뒤 필요한 값만 선택한다. table, card, list마다 반복 단위가 다르므로 구조별 selector를 분리해야 한다. 중간 변수의 길이와 타입이 맞아야 뒤의 필터링, 저장, 요약이 모두 맞는다.

### 채점 포인트

- 입력 파일을 환경 셀의 헬퍼로 읽어 코랩과 로컬에서 모두 동작하는가.
- 반복 단위와 selector 또는 파일 경로가 fixture 구조와 정확히 대응하는가.
- 숫자, 날짜, 상대 경로처럼 후처리가 필요한 값을 그대로 두지 않았는가.
- 출력 형태가 문제 요구사항과 일치하는가.

### 자주 보이는 오답

- 이전 문제 selector를 그대로 복사해 현재 HTML 구조와 맞지 않는다.
- 문자열 숫자를 변환하지 않고 비교한다.
- 결과는 비슷하지만 중간 변수명이 불분명해 최종 미션에서 재사용하기 어렵다.

---

## 문제 13 정답 — 통합 요약 만들기

~~~python
summary = {'students': len(records), 'urgent_feedback': len(urgent), 'pending_todos': counts.get('pending', 0)}
print(summary)
~~~

### 왜 이 코드가 정답인지

이 답안은 문제에서 요구한 입력을 먼저 안정적인 자료구조로 바꾼 뒤 필요한 값만 선택한다. table, card, list마다 반복 단위가 다르므로 구조별 selector를 분리해야 한다. 중간 변수의 길이와 타입이 맞아야 뒤의 필터링, 저장, 요약이 모두 맞는다.

### 채점 포인트

- 입력 파일을 환경 셀의 헬퍼로 읽어 코랩과 로컬에서 모두 동작하는가.
- 반복 단위와 selector 또는 파일 경로가 fixture 구조와 정확히 대응하는가.
- 숫자, 날짜, 상대 경로처럼 후처리가 필요한 값을 그대로 두지 않았는가.
- 출력 형태가 문제 요구사항과 일치하는가.

### 자주 보이는 오답

- 이전 문제 selector를 그대로 복사해 현재 HTML 구조와 맞지 않는다.
- 문자열 숫자를 변환하지 않고 비교한다.
- 결과는 비슷하지만 중간 변수명이 불분명해 최종 미션에서 재사용하기 어렵다.

---

## 문제 14 정답 — 정렬된 학생 목록 만들기

~~~python
top5 = sorted(records, key=lambda row: row['progress_num'], reverse=True)[:5]
print([(row['student'], row['progress_num']) for row in top5])
~~~

### 왜 이 코드가 정답인지

이 답안은 문제에서 요구한 입력을 먼저 안정적인 자료구조로 바꾼 뒤 필요한 값만 선택한다. table, card, list마다 반복 단위가 다르므로 구조별 selector를 분리해야 한다. 중간 변수의 길이와 타입이 맞아야 뒤의 필터링, 저장, 요약이 모두 맞는다.

### 채점 포인트

- 입력 파일을 환경 셀의 헬퍼로 읽어 코랩과 로컬에서 모두 동작하는가.
- 반복 단위와 selector 또는 파일 경로가 fixture 구조와 정확히 대응하는가.
- 숫자, 날짜, 상대 경로처럼 후처리가 필요한 값을 그대로 두지 않았는가.
- 출력 형태가 문제 요구사항과 일치하는가.

### 자주 보이는 오답

- 이전 문제 selector를 그대로 복사해 현재 HTML 구조와 맞지 않는다.
- 문자열 숫자를 변환하지 않고 비교한다.
- 결과는 비슷하지만 중간 변수명이 불분명해 최종 미션에서 재사용하기 어렵다.

---

## 문제 15 정답 — 통합 CSV 저장하기

~~~python
with open('lesson03_dashboard.csv', 'w', newline='', encoding='utf-8') as f:
    writer = csv.DictWriter(f, fieldnames=['student', 'course', 'progress_num', 'status'])
    writer.writeheader()
    for row in records:
        writer.writerow({'student': row['student'], 'course': row['course'], 'progress_num': row['progress_num'], 'status': row['status']})
print('saved:', 'lesson03_dashboard.csv', len(records))
~~~

### 왜 이 코드가 정답인지

이 답안은 문제에서 요구한 입력을 먼저 안정적인 자료구조로 바꾼 뒤 필요한 값만 선택한다. table, card, list마다 반복 단위가 다르므로 구조별 selector를 분리해야 한다. 중간 변수의 길이와 타입이 맞아야 뒤의 필터링, 저장, 요약이 모두 맞는다.

### 채점 포인트

- 입력 파일을 환경 셀의 헬퍼로 읽어 코랩과 로컬에서 모두 동작하는가.
- 반복 단위와 selector 또는 파일 경로가 fixture 구조와 정확히 대응하는가.
- 숫자, 날짜, 상대 경로처럼 후처리가 필요한 값을 그대로 두지 않았는가.
- 출력 형태가 문제 요구사항과 일치하는가.

### 자주 보이는 오답

- 이전 문제 selector를 그대로 복사해 현재 HTML 구조와 맞지 않는다.
- 문자열 숫자를 변환하지 않고 비교한다.
- 결과는 비슷하지만 중간 변수명이 불분명해 최종 미션에서 재사용하기 어렵다.

---

### 보강 설명 1

레슨 03 정답 해설은 실행 결과보다 과정 추적이 중요하다. 입력 파일, 반복 단위, selector 또는 경로, 변환 규칙을 분리하면 오류 위치를 빠르게 찾을 수 있다.

### 보강 설명 2

수업 중에는 학생에게 완성 코드를 먼저 보여주지 말고 HTML 구조나 CSV 헤더를 읽게 한다. 구조를 말로 설명할 수 있으면 코드는 짧아진다.

### 보강 설명 3

운영형 자동화는 다음 주에도 다시 실행되어야 한다. 그래서 파일명, URL, 저장 경로, 로그, 요약 문장을 코드 안에서 일관되게 남긴다.

### 보강 설명 4

실제 사이트로 확장할 때는 약관, robots.txt, 요청 간격, 개인정보 여부를 먼저 확인한다. 이 코스의 초반 fixture는 그 안전 습관을 만들기 위한 장치다.

### 보강 설명 5

레슨 03 정답 해설은 실행 결과보다 과정 추적이 중요하다. 입력 파일, 반복 단위, selector 또는 경로, 변환 규칙을 분리하면 오류 위치를 빠르게 찾을 수 있다.

### 보강 설명 6

수업 중에는 학생에게 완성 코드를 먼저 보여주지 말고 HTML 구조나 CSV 헤더를 읽게 한다. 구조를 말로 설명할 수 있으면 코드는 짧아진다.

### 보강 설명 7

운영형 자동화는 다음 주에도 다시 실행되어야 한다. 그래서 파일명, URL, 저장 경로, 로그, 요약 문장을 코드 안에서 일관되게 남긴다.

### 보강 설명 8

실제 사이트로 확장할 때는 약관, robots.txt, 요청 간격, 개인정보 여부를 먼저 확인한다. 이 코스의 초반 fixture는 그 안전 습관을 만들기 위한 장치다.

### 보강 설명 9

레슨 03 정답 해설은 실행 결과보다 과정 추적이 중요하다. 입력 파일, 반복 단위, selector 또는 경로, 변환 규칙을 분리하면 오류 위치를 빠르게 찾을 수 있다.

### 보강 설명 10

수업 중에는 학생에게 완성 코드를 먼저 보여주지 말고 HTML 구조나 CSV 헤더를 읽게 한다. 구조를 말로 설명할 수 있으면 코드는 짧아진다.

### 보강 설명 11

운영형 자동화는 다음 주에도 다시 실행되어야 한다. 그래서 파일명, URL, 저장 경로, 로그, 요약 문장을 코드 안에서 일관되게 남긴다.

### 보강 설명 12

실제 사이트로 확장할 때는 약관, robots.txt, 요청 간격, 개인정보 여부를 먼저 확인한다. 이 코스의 초반 fixture는 그 안전 습관을 만들기 위한 장치다.

### 보강 설명 13

레슨 03 정답 해설은 실행 결과보다 과정 추적이 중요하다. 입력 파일, 반복 단위, selector 또는 경로, 변환 규칙을 분리하면 오류 위치를 빠르게 찾을 수 있다.

### 보강 설명 14

수업 중에는 학생에게 완성 코드를 먼저 보여주지 말고 HTML 구조나 CSV 헤더를 읽게 한다. 구조를 말로 설명할 수 있으면 코드는 짧아진다.

### 보강 설명 15

운영형 자동화는 다음 주에도 다시 실행되어야 한다. 그래서 파일명, URL, 저장 경로, 로그, 요약 문장을 코드 안에서 일관되게 남긴다.

### 보강 설명 16

실제 사이트로 확장할 때는 약관, robots.txt, 요청 간격, 개인정보 여부를 먼저 확인한다. 이 코스의 초반 fixture는 그 안전 습관을 만들기 위한 장치다.

### 보강 설명 17

레슨 03 정답 해설은 실행 결과보다 과정 추적이 중요하다. 입력 파일, 반복 단위, selector 또는 경로, 변환 규칙을 분리하면 오류 위치를 빠르게 찾을 수 있다.

### 보강 설명 18

수업 중에는 학생에게 완성 코드를 먼저 보여주지 말고 HTML 구조나 CSV 헤더를 읽게 한다. 구조를 말로 설명할 수 있으면 코드는 짧아진다.
