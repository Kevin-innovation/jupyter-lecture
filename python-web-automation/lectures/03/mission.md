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

HTML 테이블과 리스트 데이터 정리 레슨의 학생용 문제 노트북이다. 강의 노트북을 먼저 실행한 뒤 빈칸을 직접 채운다.

## 통과 기준

- 총 15문제 중 12문제 이상 정상 출력이면 통과.
- 문제 1~5는 구조 확인, 6~10은 반복 추출, 11~15는 집계와 저장이다.
- 정답값은 적지 않는다. 출력 형태와 HTML/CSV 구조를 보고 직접 판단한다.

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

## 문제 1 — 대시보드 제목 읽기

HTML 표, 카드, 리스트 중 필요한 구조를 읽어 값을 정리한다.

**기대 결과 형태**: 요구한 값이 리스트, 딕셔너리 또는 CSV 저장 결과로 출력된다.

**빈칸 힌트**: HTML 구조, CSV 헤더, 이전 예제를 확인한 뒤 ____ 부분을 직접 채운다.

~~~python
dashboard_html = ____(____)
soup = ____(dashboard_html, 'html.parser')
print(soup.____('____').text.strip())
~~~

---

## 문제 2 — 테이블 헤더 추출하기

HTML 표, 카드, 리스트 중 필요한 구조를 읽어 값을 정리한다.

**기대 결과 형태**: 요구한 값이 리스트, 딕셔너리 또는 CSV 저장 결과로 출력된다.

**빈칸 힌트**: HTML 구조, CSV 헤더, 이전 예제를 확인한 뒤 ____ 부분을 직접 채운다.

~~~python
headers = [th.text.strip() for th in soup.select('____')]
print(headers)
~~~

---

## 문제 3 — 테이블 행 개수 세기

HTML 표, 카드, 리스트 중 필요한 구조를 읽어 값을 정리한다.

**기대 결과 형태**: 요구한 값이 리스트, 딕셔너리 또는 CSV 저장 결과로 출력된다.

**빈칸 힌트**: HTML 구조, CSV 헤더, 이전 예제를 확인한 뒤 ____ 부분을 직접 채운다.

~~~python
rows = soup.select('____')
print('rows:', ____)
~~~

---

## 문제 4 — 첫 행 딕셔너리 만들기

HTML 표, 카드, 리스트 중 필요한 구조를 읽어 값을 정리한다.

**기대 결과 형태**: 요구한 값이 리스트, 딕셔너리 또는 CSV 저장 결과로 출력된다.

**빈칸 힌트**: HTML 구조, CSV 헤더, 이전 예제를 확인한 뒤 ____ 부분을 직접 채운다.

~~~python
cells = [td.text.strip() for td in rows[____].select('____')]
record = dict(zip(____, ____))
print(record)
~~~

---

## 문제 5 — 전체 테이블 리스트 만들기

HTML 표, 카드, 리스트 중 필요한 구조를 읽어 값을 정리한다.

**기대 결과 형태**: 요구한 값이 리스트, 딕셔너리 또는 CSV 저장 결과로 출력된다.

**빈칸 힌트**: HTML 구조, CSV 헤더, 이전 예제를 확인한 뒤 ____ 부분을 직접 채운다.

~~~python
records = []
for tr in rows:
    cells = [td.text.strip() for td in tr.select('____')]
    records.append(dict(zip(____, ____)))
print(records[0])
print(len(records))
~~~

---

## 문제 6 — 진도율 숫자 변환하기

HTML 표, 카드, 리스트 중 필요한 구조를 읽어 값을 정리한다.

**기대 결과 형태**: 요구한 값이 리스트, 딕셔너리 또는 CSV 저장 결과로 출력된다.

**빈칸 힌트**: HTML 구조, CSV 헤더, 이전 예제를 확인한 뒤 ____ 부분을 직접 채운다.

~~~python
for row in records:
    row['progress_num'] = ____(row['____'])
print(records[0]['student'], records[0]['____'])
~~~

---

## 문제 7 — 완료 기준 학생 필터링

HTML 표, 카드, 리스트 중 필요한 구조를 읽어 값을 정리한다.

**기대 결과 형태**: 요구한 값이 리스트, 딕셔너리 또는 CSV 저장 결과로 출력된다.

**빈칸 힌트**: HTML 구조, CSV 헤더, 이전 예제를 확인한 뒤 ____ 부분을 직접 채운다.

~~~python
done_students = [row['student'] for row in records if row['____'] >= ____]
print(done_students)
~~~

---

## 문제 8 — 코스별 평균 진도 계산

HTML 표, 카드, 리스트 중 필요한 구조를 읽어 값을 정리한다.

**기대 결과 형태**: 요구한 값이 리스트, 딕셔너리 또는 CSV 저장 결과로 출력된다.

**빈칸 힌트**: HTML 구조, CSV 헤더, 이전 예제를 확인한 뒤 ____ 부분을 직접 채운다.

~~~python
summary = {}
for row in records:
    key = row['____']
    summary.setdefault(key, {'total': 0, 'count': 0})
    summary[key]['total'] += row['____']
    summary[key]['count'] += 1
averages = {k: v['total'] / v['count'] for k, v in summary.items()}
print(averages)
~~~

---

## 문제 9 — 피드백 카드 읽기

HTML 표, 카드, 리스트 중 필요한 구조를 읽어 값을 정리한다.

**기대 결과 형태**: 요구한 값이 리스트, 딕셔너리 또는 CSV 저장 결과로 출력된다.

**빈칸 힌트**: HTML 구조, CSV 헤더, 이전 예제를 확인한 뒤 ____ 부분을 직접 채운다.

~~~python
feedback_soup = BeautifulSoup(load_text('____'), 'html.parser')
cards = feedback_soup.select('____')
print(len(cards))
print(cards[0].select_one('____').text.strip())
~~~

---

## 문제 10 — 긴급 카드만 필터링

HTML 표, 카드, 리스트 중 필요한 구조를 읽어 값을 정리한다.

**기대 결과 형태**: 요구한 값이 리스트, 딕셔너리 또는 CSV 저장 결과로 출력된다.

**빈칸 힌트**: HTML 구조, CSV 헤더, 이전 예제를 확인한 뒤 ____ 부분을 직접 채운다.

~~~python
urgent = []
for card in cards:
    if card['____'] == ____:
        urgent.append(card.select_one('____').text.strip())
print(urgent)
~~~

---

## 문제 11 — todo 상태 세기

HTML 표, 카드, 리스트 중 필요한 구조를 읽어 값을 정리한다.

**기대 결과 형태**: 요구한 값이 리스트, 딕셔너리 또는 CSV 저장 결과로 출력된다.

**빈칸 힌트**: HTML 구조, CSV 헤더, 이전 예제를 확인한 뒤 ____ 부분을 직접 채운다.

~~~python
todo_soup = BeautifulSoup(load_text('____'), 'html.parser')
counts = {}
for item in todo_soup.select('____'):
    status = item['____']
    counts[status] = counts.get(status, 0) + 1
print(counts)
~~~

---

## 문제 12 — 루브릭 CSV 읽기

HTML 표, 카드, 리스트 중 필요한 구조를 읽어 값을 정리한다.

**기대 결과 형태**: 요구한 값이 리스트, 딕셔너리 또는 CSV 저장 결과로 출력된다.

**빈칸 힌트**: HTML 구조, CSV 헤더, 이전 예제를 확인한 뒤 ____ 부분을 직접 채운다.

~~~python
rubrics = list(csv.DictReader(load_text('____').splitlines()))
for row in rubrics:
    print(row['____'], row['____'])
~~~

---

## 문제 13 — 통합 요약 만들기

HTML 표, 카드, 리스트 중 필요한 구조를 읽어 값을 정리한다.

**기대 결과 형태**: 요구한 값이 리스트, 딕셔너리 또는 CSV 저장 결과로 출력된다.

**빈칸 힌트**: HTML 구조, CSV 헤더, 이전 예제를 확인한 뒤 ____ 부분을 직접 채운다.

~~~python
summary = {'students': ____, 'urgent_feedback': ____, 'pending_todos': counts.get(____, 0)}
print(summary)
~~~

---

## 문제 14 — 정렬된 학생 목록 만들기

HTML 표, 카드, 리스트 중 필요한 구조를 읽어 값을 정리한다.

**기대 결과 형태**: 요구한 값이 리스트, 딕셔너리 또는 CSV 저장 결과로 출력된다.

**빈칸 힌트**: HTML 구조, CSV 헤더, 이전 예제를 확인한 뒤 ____ 부분을 직접 채운다.

~~~python
top5 = sorted(records, key=lambda row: row['____'], reverse=____)[:____]
print([(row['student'], row['progress_num']) for row in top5])
~~~

---

## 문제 15 — 통합 CSV 저장하기

HTML 표, 카드, 리스트 중 필요한 구조를 읽어 값을 정리한다.

**기대 결과 형태**: 요구한 값이 리스트, 딕셔너리 또는 CSV 저장 결과로 출력된다.

**빈칸 힌트**: HTML 구조, CSV 헤더, 이전 예제를 확인한 뒤 ____ 부분을 직접 채운다.

~~~python
with open('lesson03_dashboard.csv', 'w', newline='', encoding='utf-8') as f:
    writer = csv.DictWriter(f, fieldnames=['student', 'course', 'progress_num', 'status'])
    writer.____()
    for row in records:
        writer.writerow({'student': row['student'], 'course': row['course'], 'progress_num': row['progress_num'], 'status': row['____']})
print('saved:', 'lesson03_dashboard.csv', len(records))
~~~

---

### 보강 설명 1

레슨 03 실습 문제은 실행 결과보다 과정 추적이 중요하다. 입력 파일, 반복 단위, selector 또는 경로, 변환 규칙을 분리하면 오류 위치를 빠르게 찾을 수 있다.

### 보강 설명 2

수업 중에는 학생에게 완성 코드를 먼저 보여주지 말고 HTML 구조나 CSV 헤더를 읽게 한다. 구조를 말로 설명할 수 있으면 코드는 짧아진다.

### 보강 설명 3

운영형 자동화는 다음 주에도 다시 실행되어야 한다. 그래서 파일명, URL, 저장 경로, 로그, 요약 문장을 코드 안에서 일관되게 남긴다.

### 보강 설명 4

실제 사이트로 확장할 때는 약관, robots.txt, 요청 간격, 개인정보 여부를 먼저 확인한다. 이 코스의 초반 fixture는 그 안전 습관을 만들기 위한 장치다.

### 보강 설명 5

레슨 03 실습 문제은 실행 결과보다 과정 추적이 중요하다. 입력 파일, 반복 단위, selector 또는 경로, 변환 규칙을 분리하면 오류 위치를 빠르게 찾을 수 있다.

### 보강 설명 6

수업 중에는 학생에게 완성 코드를 먼저 보여주지 말고 HTML 구조나 CSV 헤더를 읽게 한다. 구조를 말로 설명할 수 있으면 코드는 짧아진다.

### 보강 설명 7

운영형 자동화는 다음 주에도 다시 실행되어야 한다. 그래서 파일명, URL, 저장 경로, 로그, 요약 문장을 코드 안에서 일관되게 남긴다.

### 보강 설명 8

실제 사이트로 확장할 때는 약관, robots.txt, 요청 간격, 개인정보 여부를 먼저 확인한다. 이 코스의 초반 fixture는 그 안전 습관을 만들기 위한 장치다.

### 보강 설명 9

레슨 03 실습 문제은 실행 결과보다 과정 추적이 중요하다. 입력 파일, 반복 단위, selector 또는 경로, 변환 규칙을 분리하면 오류 위치를 빠르게 찾을 수 있다.

### 보강 설명 10

수업 중에는 학생에게 완성 코드를 먼저 보여주지 말고 HTML 구조나 CSV 헤더를 읽게 한다. 구조를 말로 설명할 수 있으면 코드는 짧아진다.

### 보강 설명 11

운영형 자동화는 다음 주에도 다시 실행되어야 한다. 그래서 파일명, URL, 저장 경로, 로그, 요약 문장을 코드 안에서 일관되게 남긴다.

### 보강 설명 12

실제 사이트로 확장할 때는 약관, robots.txt, 요청 간격, 개인정보 여부를 먼저 확인한다. 이 코스의 초반 fixture는 그 안전 습관을 만들기 위한 장치다.

### 보강 설명 13

레슨 03 실습 문제은 실행 결과보다 과정 추적이 중요하다. 입력 파일, 반복 단위, selector 또는 경로, 변환 규칙을 분리하면 오류 위치를 빠르게 찾을 수 있다.

### 보강 설명 14

수업 중에는 학생에게 완성 코드를 먼저 보여주지 말고 HTML 구조나 CSV 헤더를 읽게 한다. 구조를 말로 설명할 수 있으면 코드는 짧아진다.

### 보강 설명 15

운영형 자동화는 다음 주에도 다시 실행되어야 한다. 그래서 파일명, URL, 저장 경로, 로그, 요약 문장을 코드 안에서 일관되게 남긴다.

### 보강 설명 16

실제 사이트로 확장할 때는 약관, robots.txt, 요청 간격, 개인정보 여부를 먼저 확인한다. 이 코스의 초반 fixture는 그 안전 습관을 만들기 위한 장치다.

### 보강 설명 17

레슨 03 실습 문제은 실행 결과보다 과정 추적이 중요하다. 입력 파일, 반복 단위, selector 또는 경로, 변환 규칙을 분리하면 오류 위치를 빠르게 찾을 수 있다.

### 보강 설명 18

수업 중에는 학생에게 완성 코드를 먼저 보여주지 말고 HTML 구조나 CSV 헤더를 읽게 한다. 구조를 말로 설명할 수 있으면 코드는 짧아진다.

### 보강 설명 19

운영형 자동화는 다음 주에도 다시 실행되어야 한다. 그래서 파일명, URL, 저장 경로, 로그, 요약 문장을 코드 안에서 일관되게 남긴다.

### 보강 설명 20

실제 사이트로 확장할 때는 약관, robots.txt, 요청 간격, 개인정보 여부를 먼저 확인한다. 이 코스의 초반 fixture는 그 안전 습관을 만들기 위한 장치다.

### 보강 설명 21

레슨 03 실습 문제은 실행 결과보다 과정 추적이 중요하다. 입력 파일, 반복 단위, selector 또는 경로, 변환 규칙을 분리하면 오류 위치를 빠르게 찾을 수 있다.

### 보강 설명 22

수업 중에는 학생에게 완성 코드를 먼저 보여주지 말고 HTML 구조나 CSV 헤더를 읽게 한다. 구조를 말로 설명할 수 있으면 코드는 짧아진다.
