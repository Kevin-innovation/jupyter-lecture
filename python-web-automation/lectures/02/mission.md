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

# 레슨 02 — 실습 문제

URL 파라미터와 페이지네이션 레슨의 학생용 문제 노트북이다. 강의 노트북을 먼저 실행한 뒤 빈칸을 직접 채운다.

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
    DATA_BASE = 'https://raw.githubusercontent.com/Kevin-innovation/jupyter-lecture/main/python-web-automation/lectures/02/data'
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

## 문제 1 — 검색 URL 분해하기

지시된 값을 코드로 추출한다.

**기대 결과 형태**: 요구한 값이 한 줄 또는 리스트/딕셔너리 형태로 출력된다.

**빈칸 힌트**: HTML 구조, CSV 헤더, 이전 예제를 확인한 뒤 ____ 부분을 직접 채운다.

~~~python
sample_url = 'https://example.com/library/search?q=python&category=all&page=2'
parsed = ____(sample_url)
params = ____(parsed.query)
print(parsed.____)
print(params['____'][0], params['____'][0], params['____'][0])
~~~

---

## 문제 2 — 쿼리 문자열 만들기

지시된 값을 코드로 추출한다.

**기대 결과 형태**: 요구한 값이 한 줄 또는 리스트/딕셔너리 형태로 출력된다.

**빈칸 힌트**: HTML 구조, CSV 헤더, 이전 예제를 확인한 뒤 ____ 부분을 직접 채운다.

~~~python
base_url = 'https://example.com/library/search'
query = {'q': ____, 'category': ____, 'page': ____}
url = base_url + '?' + ____(query)
print(url)
~~~

---

## 문제 3 — 첫 페이지 HTML 읽기

지시된 값을 코드로 추출한다.

**기대 결과 형태**: 요구한 값이 한 줄 또는 리스트/딕셔너리 형태로 출력된다.

**빈칸 힌트**: HTML 구조, CSV 헤더, 이전 예제를 확인한 뒤 ____ 부분을 직접 채운다.

~~~python
html_text = ____(____)
soup = ____(html_text, 'html.parser')
print(soup.____('____').text.strip())
~~~

---

## 문제 4 — 결과 카드 개수 세기

지시된 값을 코드로 추출한다.

**기대 결과 형태**: 요구한 값이 한 줄 또는 리스트/딕셔너리 형태로 출력된다.

**빈칸 힌트**: HTML 구조, CSV 헤더, 이전 예제를 확인한 뒤 ____ 부분을 직접 채운다.

~~~python
cards = soup.____('____')
print('cards:', ____)
~~~

---

## 문제 5 — 첫 카드 제목과 href 읽기

지시된 값을 코드로 추출한다.

**기대 결과 형태**: 요구한 값이 한 줄 또는 리스트/딕셔너리 형태로 출력된다.

**빈칸 힌트**: HTML 구조, CSV 헤더, 이전 예제를 확인한 뒤 ____ 부분을 직접 채운다.

~~~python
first = cards[____]
title = first.____('____').text.strip()
href = first.____('____')['____']
print(title)
print(href)
~~~

---

## 문제 6 — 상대 URL을 절대 URL로 바꾸기

지시된 값을 코드로 추출한다.

**기대 결과 형태**: 요구한 값이 한 줄 또는 리스트/딕셔너리 형태로 출력된다.

**빈칸 힌트**: HTML 구조, CSV 헤더, 이전 예제를 확인한 뒤 ____ 부분을 직접 채운다.

~~~python
absolute = ____('https://example.com', ____)
print(absolute)
~~~

---

## 문제 7 — 카드 하나를 딕셔너리로 만들기

지시된 값을 코드로 추출한다.

**기대 결과 형태**: 요구한 값이 한 줄 또는 리스트/딕셔너리 형태로 출력된다.

**빈칸 힌트**: HTML 구조, CSV 헤더, 이전 예제를 확인한 뒤 ____ 부분을 직접 채운다.

~~~python
item = {'title': first.select_one('____').text.strip(), 'category': first['____'], 'date': first.select_one('____')['____'], 'views': ____(first.select_one('____').text), 'url': urljoin('https://example.com', first.select_one('____')['____'])}
print(item)
~~~

---

## 문제 8 — 한 페이지 결과 리스트 만들기

지시된 값을 코드로 추출한다.

**기대 결과 형태**: 요구한 값이 한 줄 또는 리스트/딕셔너리 형태로 출력된다.

**빈칸 힌트**: HTML 구조, CSV 헤더, 이전 예제를 확인한 뒤 ____ 부분을 직접 채운다.

~~~python
results = []
for card in cards:
    results.append({'title': card.select_one('____').text.strip(), 'category': card['____'], 'page': int(card['____']), 'views': ____(card.select_one('____').text)})
print(results[0])
print(len(results))
~~~

---

## 문제 9 — 페이지 번호로 파일명 만들기

지시된 값을 코드로 추출한다.

**기대 결과 형태**: 요구한 값이 한 줄 또는 리스트/딕셔너리 형태로 출력된다.

**빈칸 힌트**: HTML 구조, CSV 헤더, 이전 예제를 확인한 뒤 ____ 부분을 직접 채운다.

~~~python
def page_filename(page):
    return f'_____{page}.____'
for page in [1, 2, 3]:
    print(____(page))
~~~

---

## 문제 10 — 3페이지 전체 순회하기

지시된 값을 코드로 추출한다.

**기대 결과 형태**: 요구한 값이 한 줄 또는 리스트/딕셔너리 형태로 출력된다.

**빈칸 힌트**: HTML 구조, CSV 헤더, 이전 예제를 확인한 뒤 ____ 부분을 직접 채운다.

~~~python
all_results = []
for page in range(____, ____):
    page_soup = BeautifulSoup(load_text(____(page)), 'html.parser')
    for card in page_soup.select('____'):
        all_results.append(card.select_one('____').text.strip())
print(len(all_results))
print(all_results[-1])
~~~

---

## 문제 11 — 다음 페이지 링크 찾기

지시된 값을 코드로 추출한다.

**기대 결과 형태**: 요구한 값이 한 줄 또는 리스트/딕셔너리 형태로 출력된다.

**빈칸 힌트**: HTML 구조, CSV 헤더, 이전 예제를 확인한 뒤 ____ 부분을 직접 채운다.

~~~python
next_href = soup.select_one('____')['____']
next_query = ____(____(next_href).query)
print(next_href)
print(next_query['____'][0])
~~~

---

## 문제 12 — 조회수 1000 이상 필터링

지시된 값을 코드로 추출한다.

**기대 결과 형태**: 요구한 값이 한 줄 또는 리스트/딕셔너리 형태로 출력된다.

**빈칸 힌트**: HTML 구조, CSV 헤더, 이전 예제를 확인한 뒤 ____ 부분을 직접 채운다.

~~~python
rich = []
for page in range(1, 4):
    page_soup = BeautifulSoup(load_text(page_filename(page)), 'html.parser')
    for card in page_soup.select('article.result-card'):
        views = ____(card.select_one('____').text)
        if views >= ____:
            rich.append(card.select_one('____').text.strip())
print(rich)
~~~

---

## 문제 13 — 카테고리별 개수 세기

지시된 값을 코드로 추출한다.

**기대 결과 형태**: 요구한 값이 한 줄 또는 리스트/딕셔너리 형태로 출력된다.

**빈칸 힌트**: HTML 구조, CSV 헤더, 이전 예제를 확인한 뒤 ____ 부분을 직접 채운다.

~~~python
counts = {}
for page in range(1, 4):
    page_soup = BeautifulSoup(load_text(page_filename(page)), 'html.parser')
    for card in page_soup.select('article.result-card'):
        key = card['____']
        counts[key] = counts.get(key, ____) + ____
print(counts)
~~~

---

## 문제 14 — targets CSV 읽기

지시된 값을 코드로 추출한다.

**기대 결과 형태**: 요구한 값이 한 줄 또는 리스트/딕셔너리 형태로 출력된다.

**빈칸 힌트**: HTML 구조, CSV 헤더, 이전 예제를 확인한 뒤 ____ 부분을 직접 채운다.

~~~python
target_rows = list(csv.DictReader(load_text('____').splitlines()))
for row in target_rows:
    print(row['____'], row['____'])
~~~

---

## 문제 15 — 검색 결과 CSV 저장하기

지시된 값을 코드로 추출한다.

**기대 결과 형태**: 요구한 값이 한 줄 또는 리스트/딕셔너리 형태로 출력된다.

**빈칸 힌트**: HTML 구조, CSV 헤더, 이전 예제를 확인한 뒤 ____ 부분을 직접 채운다.

~~~python
rows = []
for page in range(1, 4):
    page_soup = BeautifulSoup(load_text(page_filename(page)), 'html.parser')
    for card in page_soup.select('article.result-card'):
        rows.append({'title': card.select_one('____').text.strip(), 'category': card['____'], 'views': ____(card.select_one('____').text)})
with open('lesson02_results.csv', 'w', newline='', encoding='utf-8') as f:
    writer = csv.DictWriter(f, fieldnames=['title', 'category', 'views'])
    writer.____()
    writer.____(rows)
print('saved:', 'lesson02_results.csv', len(rows))
~~~

---

### 보강 설명 1

레슨 02 실습 문제은 실행 결과보다 과정 추적이 중요하다. 입력 파일, 반복 단위, selector 또는 경로, 변환 규칙을 분리하면 오류 위치를 빠르게 찾을 수 있다.

### 보강 설명 2

수업 중에는 학생에게 완성 코드를 먼저 보여주지 말고 HTML 구조나 CSV 헤더를 읽게 한다. 구조를 말로 설명할 수 있으면 코드는 짧아진다.

### 보강 설명 3

운영형 자동화는 다음 주에도 다시 실행되어야 한다. 그래서 파일명, URL, 저장 경로, 로그, 요약 문장을 코드 안에서 일관되게 남긴다.

### 보강 설명 4

실제 사이트로 확장할 때는 약관, robots.txt, 요청 간격, 개인정보 여부를 먼저 확인한다. 이 코스의 초반 fixture는 그 안전 습관을 만들기 위한 장치다.

### 보강 설명 5

레슨 02 실습 문제은 실행 결과보다 과정 추적이 중요하다. 입력 파일, 반복 단위, selector 또는 경로, 변환 규칙을 분리하면 오류 위치를 빠르게 찾을 수 있다.

### 보강 설명 6

수업 중에는 학생에게 완성 코드를 먼저 보여주지 말고 HTML 구조나 CSV 헤더를 읽게 한다. 구조를 말로 설명할 수 있으면 코드는 짧아진다.

### 보강 설명 7

운영형 자동화는 다음 주에도 다시 실행되어야 한다. 그래서 파일명, URL, 저장 경로, 로그, 요약 문장을 코드 안에서 일관되게 남긴다.

### 보강 설명 8

실제 사이트로 확장할 때는 약관, robots.txt, 요청 간격, 개인정보 여부를 먼저 확인한다. 이 코스의 초반 fixture는 그 안전 습관을 만들기 위한 장치다.

### 보강 설명 9

레슨 02 실습 문제은 실행 결과보다 과정 추적이 중요하다. 입력 파일, 반복 단위, selector 또는 경로, 변환 규칙을 분리하면 오류 위치를 빠르게 찾을 수 있다.

### 보강 설명 10

수업 중에는 학생에게 완성 코드를 먼저 보여주지 말고 HTML 구조나 CSV 헤더를 읽게 한다. 구조를 말로 설명할 수 있으면 코드는 짧아진다.

### 보강 설명 11

운영형 자동화는 다음 주에도 다시 실행되어야 한다. 그래서 파일명, URL, 저장 경로, 로그, 요약 문장을 코드 안에서 일관되게 남긴다.

### 보강 설명 12

실제 사이트로 확장할 때는 약관, robots.txt, 요청 간격, 개인정보 여부를 먼저 확인한다. 이 코스의 초반 fixture는 그 안전 습관을 만들기 위한 장치다.

### 보강 설명 13

레슨 02 실습 문제은 실행 결과보다 과정 추적이 중요하다. 입력 파일, 반복 단위, selector 또는 경로, 변환 규칙을 분리하면 오류 위치를 빠르게 찾을 수 있다.

### 보강 설명 14

수업 중에는 학생에게 완성 코드를 먼저 보여주지 말고 HTML 구조나 CSV 헤더를 읽게 한다. 구조를 말로 설명할 수 있으면 코드는 짧아진다.

### 보강 설명 15

운영형 자동화는 다음 주에도 다시 실행되어야 한다. 그래서 파일명, URL, 저장 경로, 로그, 요약 문장을 코드 안에서 일관되게 남긴다.

### 보강 설명 16

실제 사이트로 확장할 때는 약관, robots.txt, 요청 간격, 개인정보 여부를 먼저 확인한다. 이 코스의 초반 fixture는 그 안전 습관을 만들기 위한 장치다.

### 보강 설명 17

레슨 02 실습 문제은 실행 결과보다 과정 추적이 중요하다. 입력 파일, 반복 단위, selector 또는 경로, 변환 규칙을 분리하면 오류 위치를 빠르게 찾을 수 있다.

### 보강 설명 18

수업 중에는 학생에게 완성 코드를 먼저 보여주지 말고 HTML 구조나 CSV 헤더를 읽게 한다. 구조를 말로 설명할 수 있으면 코드는 짧아진다.

### 보강 설명 19

운영형 자동화는 다음 주에도 다시 실행되어야 한다. 그래서 파일명, URL, 저장 경로, 로그, 요약 문장을 코드 안에서 일관되게 남긴다.

### 보강 설명 20

실제 사이트로 확장할 때는 약관, robots.txt, 요청 간격, 개인정보 여부를 먼저 확인한다. 이 코스의 초반 fixture는 그 안전 습관을 만들기 위한 장치다.

### 보강 설명 21

레슨 02 실습 문제은 실행 결과보다 과정 추적이 중요하다. 입력 파일, 반복 단위, selector 또는 경로, 변환 규칙을 분리하면 오류 위치를 빠르게 찾을 수 있다.
