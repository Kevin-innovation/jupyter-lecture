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

# 레슨 02 — 실습 문제 정답지

> 🔒 교사·관리자 전용. 학생에게 배포 금지.

URL 파라미터와 페이지네이션 실습 문제의 모범 답안이다. 출력값만 보지 말고 selector, 타입 변환, 저장 흐름을 같이 확인한다.

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

## 문제 1 정답 — 검색 URL 분해하기

~~~python
sample_url = 'https://example.com/library/search?q=python&category=all&page=2'
parsed = urlparse(sample_url)
params = parse_qs(parsed.query)
print(parsed.path)
print(params['q'][0], params['category'][0], params['page'][0])
~~~

### 왜 이 코드가 정답인지

이 답안은 문제에서 요구한 입력을 먼저 안정적인 자료구조로 바꾼 뒤 필요한 값만 선택한다. URL, query string, 페이지 번호와 카드 selector가 서로 연결되어야 한다. 중간 변수의 길이와 타입이 맞아야 뒤의 필터링, 저장, 요약이 모두 맞는다.

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

## 문제 2 정답 — 쿼리 문자열 만들기

~~~python
base_url = 'https://example.com/library/search'
query = {'q': 'python automation', 'category': 'course', 'page': 1}
url = base_url + '?' + urlencode(query)
print(url)
~~~

### 왜 이 코드가 정답인지

이 답안은 문제에서 요구한 입력을 먼저 안정적인 자료구조로 바꾼 뒤 필요한 값만 선택한다. URL, query string, 페이지 번호와 카드 selector가 서로 연결되어야 한다. 중간 변수의 길이와 타입이 맞아야 뒤의 필터링, 저장, 요약이 모두 맞는다.

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

## 문제 3 정답 — 첫 페이지 HTML 읽기

~~~python
html_text = load_text('search_page_1.html')
soup = BeautifulSoup(html_text, 'html.parser')
print(soup.select_one('h1').text.strip())
~~~

### 왜 이 코드가 정답인지

이 답안은 문제에서 요구한 입력을 먼저 안정적인 자료구조로 바꾼 뒤 필요한 값만 선택한다. URL, query string, 페이지 번호와 카드 selector가 서로 연결되어야 한다. 중간 변수의 길이와 타입이 맞아야 뒤의 필터링, 저장, 요약이 모두 맞는다.

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

## 문제 4 정답 — 결과 카드 개수 세기

~~~python
cards = soup.select('article.result-card')
print('cards:', len(cards))
~~~

### 왜 이 코드가 정답인지

이 답안은 문제에서 요구한 입력을 먼저 안정적인 자료구조로 바꾼 뒤 필요한 값만 선택한다. URL, query string, 페이지 번호와 카드 selector가 서로 연결되어야 한다. 중간 변수의 길이와 타입이 맞아야 뒤의 필터링, 저장, 요약이 모두 맞는다.

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

## 문제 5 정답 — 첫 카드 제목과 href 읽기

~~~python
first = cards[0]
title = first.select_one('.title a').text.strip()
href = first.select_one('.title a')['href']
print(title)
print(href)
~~~

### 왜 이 코드가 정답인지

이 답안은 문제에서 요구한 입력을 먼저 안정적인 자료구조로 바꾼 뒤 필요한 값만 선택한다. URL, query string, 페이지 번호와 카드 selector가 서로 연결되어야 한다. 중간 변수의 길이와 타입이 맞아야 뒤의 필터링, 저장, 요약이 모두 맞는다.

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

## 문제 6 정답 — 상대 URL을 절대 URL로 바꾸기

~~~python
absolute = urljoin('https://example.com', href)
print(absolute)
~~~

### 왜 이 코드가 정답인지

이 답안은 문제에서 요구한 입력을 먼저 안정적인 자료구조로 바꾼 뒤 필요한 값만 선택한다. URL, query string, 페이지 번호와 카드 selector가 서로 연결되어야 한다. 중간 변수의 길이와 타입이 맞아야 뒤의 필터링, 저장, 요약이 모두 맞는다.

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

## 문제 7 정답 — 카드 하나를 딕셔너리로 만들기

~~~python
item = {'title': first.select_one('.title a').text.strip(), 'category': first['data-category'], 'date': first.select_one('time')['datetime'], 'views': clean_int(first.select_one('.views').text), 'url': urljoin('https://example.com', first.select_one('.title a')['href'])}
print(item)
~~~

### 왜 이 코드가 정답인지

이 답안은 문제에서 요구한 입력을 먼저 안정적인 자료구조로 바꾼 뒤 필요한 값만 선택한다. URL, query string, 페이지 번호와 카드 selector가 서로 연결되어야 한다. 중간 변수의 길이와 타입이 맞아야 뒤의 필터링, 저장, 요약이 모두 맞는다.

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

## 문제 8 정답 — 한 페이지 결과 리스트 만들기

~~~python
results = []
for card in cards:
    results.append({'title': card.select_one('.title a').text.strip(), 'category': card['data-category'], 'page': int(card['data-page']), 'views': clean_int(card.select_one('.views').text)})
print(results[0])
print(len(results))
~~~

### 왜 이 코드가 정답인지

이 답안은 문제에서 요구한 입력을 먼저 안정적인 자료구조로 바꾼 뒤 필요한 값만 선택한다. URL, query string, 페이지 번호와 카드 selector가 서로 연결되어야 한다. 중간 변수의 길이와 타입이 맞아야 뒤의 필터링, 저장, 요약이 모두 맞는다.

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

## 문제 9 정답 — 페이지 번호로 파일명 만들기

~~~python
def page_filename(page):
    return f'search_page_{page}.html'
for page in [1, 2, 3]:
    print(page_filename(page))
~~~

### 왜 이 코드가 정답인지

이 답안은 문제에서 요구한 입력을 먼저 안정적인 자료구조로 바꾼 뒤 필요한 값만 선택한다. URL, query string, 페이지 번호와 카드 selector가 서로 연결되어야 한다. 중간 변수의 길이와 타입이 맞아야 뒤의 필터링, 저장, 요약이 모두 맞는다.

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

## 문제 10 정답 — 3페이지 전체 순회하기

~~~python
all_results = []
for page in range(1, 4):
    page_soup = BeautifulSoup(load_text(page_filename(page)), 'html.parser')
    for card in page_soup.select('article.result-card'):
        all_results.append(card.select_one('.title a').text.strip())
print(len(all_results))
print(all_results[-1])
~~~

### 왜 이 코드가 정답인지

이 답안은 문제에서 요구한 입력을 먼저 안정적인 자료구조로 바꾼 뒤 필요한 값만 선택한다. URL, query string, 페이지 번호와 카드 selector가 서로 연결되어야 한다. 중간 변수의 길이와 타입이 맞아야 뒤의 필터링, 저장, 요약이 모두 맞는다.

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

## 문제 11 정답 — 다음 페이지 링크 찾기

~~~python
next_href = soup.select_one('a.next')['href']
next_query = parse_qs(urlparse(next_href).query)
print(next_href)
print(next_query['page'][0])
~~~

### 왜 이 코드가 정답인지

이 답안은 문제에서 요구한 입력을 먼저 안정적인 자료구조로 바꾼 뒤 필요한 값만 선택한다. URL, query string, 페이지 번호와 카드 selector가 서로 연결되어야 한다. 중간 변수의 길이와 타입이 맞아야 뒤의 필터링, 저장, 요약이 모두 맞는다.

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

## 문제 12 정답 — 조회수 1000 이상 필터링

~~~python
rich = []
for page in range(1, 4):
    page_soup = BeautifulSoup(load_text(page_filename(page)), 'html.parser')
    for card in page_soup.select('article.result-card'):
        views = clean_int(card.select_one('.views').text)
        if views >= 1000:
            rich.append(card.select_one('.title a').text.strip())
print(rich)
~~~

### 왜 이 코드가 정답인지

이 답안은 문제에서 요구한 입력을 먼저 안정적인 자료구조로 바꾼 뒤 필요한 값만 선택한다. URL, query string, 페이지 번호와 카드 selector가 서로 연결되어야 한다. 중간 변수의 길이와 타입이 맞아야 뒤의 필터링, 저장, 요약이 모두 맞는다.

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

## 문제 13 정답 — 카테고리별 개수 세기

~~~python
counts = {}
for page in range(1, 4):
    page_soup = BeautifulSoup(load_text(page_filename(page)), 'html.parser')
    for card in page_soup.select('article.result-card'):
        key = card['data-category']
        counts[key] = counts.get(key, 0) + 1
print(counts)
~~~

### 왜 이 코드가 정답인지

이 답안은 문제에서 요구한 입력을 먼저 안정적인 자료구조로 바꾼 뒤 필요한 값만 선택한다. URL, query string, 페이지 번호와 카드 selector가 서로 연결되어야 한다. 중간 변수의 길이와 타입이 맞아야 뒤의 필터링, 저장, 요약이 모두 맞는다.

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

## 문제 14 정답 — targets CSV 읽기

~~~python
target_rows = list(csv.DictReader(load_text('search_targets.csv').splitlines()))
for row in target_rows:
    print(row['query'], row['max_pages'])
~~~

### 왜 이 코드가 정답인지

이 답안은 문제에서 요구한 입력을 먼저 안정적인 자료구조로 바꾼 뒤 필요한 값만 선택한다. URL, query string, 페이지 번호와 카드 selector가 서로 연결되어야 한다. 중간 변수의 길이와 타입이 맞아야 뒤의 필터링, 저장, 요약이 모두 맞는다.

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

## 문제 15 정답 — 검색 결과 CSV 저장하기

~~~python
rows = []
for page in range(1, 4):
    page_soup = BeautifulSoup(load_text(page_filename(page)), 'html.parser')
    for card in page_soup.select('article.result-card'):
        rows.append({'title': card.select_one('.title a').text.strip(), 'category': card['data-category'], 'views': clean_int(card.select_one('.views').text)})
with open('lesson02_results.csv', 'w', newline='', encoding='utf-8') as f:
    writer = csv.DictWriter(f, fieldnames=['title', 'category', 'views'])
    writer.writeheader()
    writer.writerows(rows)
print('saved:', 'lesson02_results.csv', len(rows))
~~~

### 왜 이 코드가 정답인지

이 답안은 문제에서 요구한 입력을 먼저 안정적인 자료구조로 바꾼 뒤 필요한 값만 선택한다. URL, query string, 페이지 번호와 카드 selector가 서로 연결되어야 한다. 중간 변수의 길이와 타입이 맞아야 뒤의 필터링, 저장, 요약이 모두 맞는다.

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

레슨 02 정답 해설은 실행 결과보다 과정 추적이 중요하다. 입력 파일, 반복 단위, selector 또는 경로, 변환 규칙을 분리하면 오류 위치를 빠르게 찾을 수 있다.

### 보강 설명 2

수업 중에는 학생에게 완성 코드를 먼저 보여주지 말고 HTML 구조나 CSV 헤더를 읽게 한다. 구조를 말로 설명할 수 있으면 코드는 짧아진다.

### 보강 설명 3

운영형 자동화는 다음 주에도 다시 실행되어야 한다. 그래서 파일명, URL, 저장 경로, 로그, 요약 문장을 코드 안에서 일관되게 남긴다.

### 보강 설명 4

실제 사이트로 확장할 때는 약관, robots.txt, 요청 간격, 개인정보 여부를 먼저 확인한다. 이 코스의 초반 fixture는 그 안전 습관을 만들기 위한 장치다.

### 보강 설명 5

레슨 02 정답 해설은 실행 결과보다 과정 추적이 중요하다. 입력 파일, 반복 단위, selector 또는 경로, 변환 규칙을 분리하면 오류 위치를 빠르게 찾을 수 있다.

### 보강 설명 6

수업 중에는 학생에게 완성 코드를 먼저 보여주지 말고 HTML 구조나 CSV 헤더를 읽게 한다. 구조를 말로 설명할 수 있으면 코드는 짧아진다.

### 보강 설명 7

운영형 자동화는 다음 주에도 다시 실행되어야 한다. 그래서 파일명, URL, 저장 경로, 로그, 요약 문장을 코드 안에서 일관되게 남긴다.

### 보강 설명 8

실제 사이트로 확장할 때는 약관, robots.txt, 요청 간격, 개인정보 여부를 먼저 확인한다. 이 코스의 초반 fixture는 그 안전 습관을 만들기 위한 장치다.

### 보강 설명 9

레슨 02 정답 해설은 실행 결과보다 과정 추적이 중요하다. 입력 파일, 반복 단위, selector 또는 경로, 변환 규칙을 분리하면 오류 위치를 빠르게 찾을 수 있다.

### 보강 설명 10

수업 중에는 학생에게 완성 코드를 먼저 보여주지 말고 HTML 구조나 CSV 헤더를 읽게 한다. 구조를 말로 설명할 수 있으면 코드는 짧아진다.

### 보강 설명 11

운영형 자동화는 다음 주에도 다시 실행되어야 한다. 그래서 파일명, URL, 저장 경로, 로그, 요약 문장을 코드 안에서 일관되게 남긴다.

### 보강 설명 12

실제 사이트로 확장할 때는 약관, robots.txt, 요청 간격, 개인정보 여부를 먼저 확인한다. 이 코스의 초반 fixture는 그 안전 습관을 만들기 위한 장치다.

### 보강 설명 13

레슨 02 정답 해설은 실행 결과보다 과정 추적이 중요하다. 입력 파일, 반복 단위, selector 또는 경로, 변환 규칙을 분리하면 오류 위치를 빠르게 찾을 수 있다.

### 보강 설명 14

수업 중에는 학생에게 완성 코드를 먼저 보여주지 말고 HTML 구조나 CSV 헤더를 읽게 한다. 구조를 말로 설명할 수 있으면 코드는 짧아진다.

### 보강 설명 15

운영형 자동화는 다음 주에도 다시 실행되어야 한다. 그래서 파일명, URL, 저장 경로, 로그, 요약 문장을 코드 안에서 일관되게 남긴다.
