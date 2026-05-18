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

> 교사·관리자 전용. 학생에게 배포 금지.

URL 파라미터와 페이지네이션 실습 문제의 모범 답안이다. 출력값만 확인하지 말고 URL 분해, selector 기준, 타입 변환, 저장 흐름을 같이 확인한다.

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

def page_filename(page):
    return f'search_page_{page}.html'

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

URL의 path와 query는 역할이 다르다. urlparse로 URL을 구조화해야 path만 따로 확인할 수 있고, parse_qs로 q, category, page를 키로 가진 딕셔너리를 얻는다. query 값은 리스트로 반환되므로 첫 값을 읽는다.

### 채점 포인트

- 입력 파일을 환경 셀의 헬퍼로 읽어 코랩과 로컬에서 모두 동작하는가.
- 반복 단위와 selector 또는 URL 처리 함수가 fixture 구조와 정확히 대응하는가.
- 숫자, 날짜, 상대 경로처럼 후처리가 필요한 값을 그대로 두지 않았는가.
- 출력 형태가 문제 요구사항과 일치하는가.

### 자주 보이는 오답

- 이전 문제 selector를 그대로 복사해 현재 구조와 맞지 않는다.
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

urlencode는 딕셔너리의 키와 값을 URL query string 규칙에 맞춰 변환한다. 검색어에 공백이 있어도 안전하게 인코딩되므로 문자열을 직접 이어 붙이는 방식보다 운영 자동화에 적합하다.

### 채점 포인트

- 입력 파일을 환경 셀의 헬퍼로 읽어 코랩과 로컬에서 모두 동작하는가.
- 반복 단위와 selector 또는 URL 처리 함수가 fixture 구조와 정확히 대응하는가.
- 숫자, 날짜, 상대 경로처럼 후처리가 필요한 값을 그대로 두지 않았는가.
- 출력 형태가 문제 요구사항과 일치하는가.

### 자주 보이는 오답

- 이전 문제 selector를 그대로 복사해 현재 구조와 맞지 않는다.
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

load_text는 코랩과 로컬 경로 차이를 숨긴다. BeautifulSoup 객체로 바꾼 뒤 select_one으로 페이지 제목 하나를 안정적으로 가져올 수 있다.

### 채점 포인트

- 입력 파일을 환경 셀의 헬퍼로 읽어 코랩과 로컬에서 모두 동작하는가.
- 반복 단위와 selector 또는 URL 처리 함수가 fixture 구조와 정확히 대응하는가.
- 숫자, 날짜, 상대 경로처럼 후처리가 필요한 값을 그대로 두지 않았는가.
- 출력 형태가 문제 요구사항과 일치하는가.

### 자주 보이는 오답

- 이전 문제 selector를 그대로 복사해 현재 구조와 맞지 않는다.
- 문자열 숫자를 변환하지 않고 비교한다.
- 결과는 비슷하지만 중간 변수명이 불분명해 최종 미션에서 재사용하기 어렵다.

---

## 문제 4 정답 — 결과 카드 개수 세기

~~~python
cards = soup.select('article.result-card')
print('cards:', len(cards))
~~~

### 왜 이 코드가 정답인지

페이지 전체에서 같은 구조가 반복될 때는 select로 리스트를 받는다. article.result-card는 다른 링크나 pagination을 제외하고 검색 결과 카드만 잡는 selector다.

### 채점 포인트

- 입력 파일을 환경 셀의 헬퍼로 읽어 코랩과 로컬에서 모두 동작하는가.
- 반복 단위와 selector 또는 URL 처리 함수가 fixture 구조와 정확히 대응하는가.
- 숫자, 날짜, 상대 경로처럼 후처리가 필요한 값을 그대로 두지 않았는가.
- 출력 형태가 문제 요구사항과 일치하는가.

### 자주 보이는 오답

- 이전 문제 selector를 그대로 복사해 현재 구조와 맞지 않는다.
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

카드 하나 안에서 다시 .title a를 찾으면 다른 카드의 링크와 섞이지 않는다. 텍스트는 정리하고, 링크 경로는 href 속성으로 읽는다.

### 채점 포인트

- 입력 파일을 환경 셀의 헬퍼로 읽어 코랩과 로컬에서 모두 동작하는가.
- 반복 단위와 selector 또는 URL 처리 함수가 fixture 구조와 정확히 대응하는가.
- 숫자, 날짜, 상대 경로처럼 후처리가 필요한 값을 그대로 두지 않았는가.
- 출력 형태가 문제 요구사항과 일치하는가.

### 자주 보이는 오답

- 이전 문제 selector를 그대로 복사해 현재 구조와 맞지 않는다.
- 문자열 숫자를 변환하지 않고 비교한다.
- 결과는 비슷하지만 중간 변수명이 불분명해 최종 미션에서 재사용하기 어렵다.

---

## 문제 6 정답 — 상대 URL을 절대 URL로 바꾸기

~~~python
absolute = urljoin('https://example.com', href)
print(absolute)
~~~

### 왜 이 코드가 정답인지

상대 링크는 저장만 해서는 어느 도메인의 경로인지 알 수 없다. urljoin은 기준 URL과 상대 경로를 올바르게 합쳐 다음 자동화 단계에서 바로 열 수 있는 URL을 만든다.

### 채점 포인트

- 입력 파일을 환경 셀의 헬퍼로 읽어 코랩과 로컬에서 모두 동작하는가.
- 반복 단위와 selector 또는 URL 처리 함수가 fixture 구조와 정확히 대응하는가.
- 숫자, 날짜, 상대 경로처럼 후처리가 필요한 값을 그대로 두지 않았는가.
- 출력 형태가 문제 요구사항과 일치하는가.

### 자주 보이는 오답

- 이전 문제 selector를 그대로 복사해 현재 구조와 맞지 않는다.
- 문자열 숫자를 변환하지 않고 비교한다.
- 결과는 비슷하지만 중간 변수명이 불분명해 최종 미션에서 재사용하기 어렵다.

---

## 문제 7 정답 — 카드 하나를 딕셔너리로 만들기

~~~python
item = {
    'title': first.select_one('.title a').text.strip(),
    'category': first['data-category'],
    'date': first.select_one('time')['datetime'],
    'views': clean_int(first.select_one('.views').text),
    'url': urljoin('https://example.com', first.select_one('.title a')['href']),
}
print(item)
~~~

### 왜 이 코드가 정답인지

웹 화면의 값은 태그 텍스트, HTML 속성, 문자열 숫자가 섞여 있다. 이 답안은 각 값의 위치에 맞는 읽기 방식을 사용하고, 저장하기 좋은 딕셔너리 구조로 통일한다.

### 채점 포인트

- 입력 파일을 환경 셀의 헬퍼로 읽어 코랩과 로컬에서 모두 동작하는가.
- 반복 단위와 selector 또는 URL 처리 함수가 fixture 구조와 정확히 대응하는가.
- 숫자, 날짜, 상대 경로처럼 후처리가 필요한 값을 그대로 두지 않았는가.
- 출력 형태가 문제 요구사항과 일치하는가.

### 자주 보이는 오답

- 이전 문제 selector를 그대로 복사해 현재 구조와 맞지 않는다.
- 문자열 숫자를 변환하지 않고 비교한다.
- 결과는 비슷하지만 중간 변수명이 불분명해 최종 미션에서 재사용하기 어렵다.

---

## 문제 8 정답 — 한 페이지 결과 리스트 만들기

~~~python
results = []
for card in cards:
    results.append({
        'title': card.select_one('.title a').text.strip(),
        'category': card['data-category'],
        'page': int(card['data-page']),
        'views': clean_int(card.select_one('.views').text),
    })
print(results[0])
print(len(results))
~~~

### 왜 이 코드가 정답인지

한 페이지 안의 카드들을 같은 딕셔너리 구조로 만들면 이후 필터링과 저장이 쉬워진다. data-page는 문자열 속성이므로 정수로 변환한다.

### 채점 포인트

- 입력 파일을 환경 셀의 헬퍼로 읽어 코랩과 로컬에서 모두 동작하는가.
- 반복 단위와 selector 또는 URL 처리 함수가 fixture 구조와 정확히 대응하는가.
- 숫자, 날짜, 상대 경로처럼 후처리가 필요한 값을 그대로 두지 않았는가.
- 출력 형태가 문제 요구사항과 일치하는가.

### 자주 보이는 오답

- 이전 문제 selector를 그대로 복사해 현재 구조와 맞지 않는다.
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

페이지 파일명 규칙을 함수로 분리하면 여러 페이지 순회 코드에서 파일명 문자열을 반복 작성하지 않아도 된다. 규칙이 바뀌어도 함수 한 곳만 수정하면 된다.

### 채점 포인트

- 입력 파일을 환경 셀의 헬퍼로 읽어 코랩과 로컬에서 모두 동작하는가.
- 반복 단위와 selector 또는 URL 처리 함수가 fixture 구조와 정확히 대응하는가.
- 숫자, 날짜, 상대 경로처럼 후처리가 필요한 값을 그대로 두지 않았는가.
- 출력 형태가 문제 요구사항과 일치하는가.

### 자주 보이는 오답

- 이전 문제 selector를 그대로 복사해 현재 구조와 맞지 않는다.
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

바깥 반복문은 페이지를 바꾸고 안쪽 반복문은 카드 단위를 처리한다. 3개 파일에 각각 9개 카드가 있으므로 전체 제목 수로 페이지 순회를 확인할 수 있다.

### 채점 포인트

- 입력 파일을 환경 셀의 헬퍼로 읽어 코랩과 로컬에서 모두 동작하는가.
- 반복 단위와 selector 또는 URL 처리 함수가 fixture 구조와 정확히 대응하는가.
- 숫자, 날짜, 상대 경로처럼 후처리가 필요한 값을 그대로 두지 않았는가.
- 출력 형태가 문제 요구사항과 일치하는가.

### 자주 보이는 오답

- 이전 문제 selector를 그대로 복사해 현재 구조와 맞지 않는다.
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

페이지네이션 링크는 화면에 보이는 숫자뿐 아니라 다음 페이지로 가는 URL을 제공한다. href에서 query string을 다시 분해하면 다음 page 값을 코드로 판단할 수 있다.

### 채점 포인트

- 입력 파일을 환경 셀의 헬퍼로 읽어 코랩과 로컬에서 모두 동작하는가.
- 반복 단위와 selector 또는 URL 처리 함수가 fixture 구조와 정확히 대응하는가.
- 숫자, 날짜, 상대 경로처럼 후처리가 필요한 값을 그대로 두지 않았는가.
- 출력 형태가 문제 요구사항과 일치하는가.

### 자주 보이는 오답

- 이전 문제 selector를 그대로 복사해 현재 구조와 맞지 않는다.
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

HTML의 조회수는 사람이 읽는 문자열이다. 숫자 비교를 하려면 clean_int로 쉼표와 한글을 제거해 정수로 바꾼 뒤 기준값 1000과 비교해야 한다.

### 채점 포인트

- 입력 파일을 환경 셀의 헬퍼로 읽어 코랩과 로컬에서 모두 동작하는가.
- 반복 단위와 selector 또는 URL 처리 함수가 fixture 구조와 정확히 대응하는가.
- 숫자, 날짜, 상대 경로처럼 후처리가 필요한 값을 그대로 두지 않았는가.
- 출력 형태가 문제 요구사항과 일치하는가.

### 자주 보이는 오답

- 이전 문제 selector를 그대로 복사해 현재 구조와 맞지 않는다.
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

카테고리는 카드의 data-category 속성에 들어 있다. 딕셔너리 집계는 기존 값이 없을 때 0에서 시작해 1씩 더하는 방식으로 구현한다.

### 채점 포인트

- 입력 파일을 환경 셀의 헬퍼로 읽어 코랩과 로컬에서 모두 동작하는가.
- 반복 단위와 selector 또는 URL 처리 함수가 fixture 구조와 정확히 대응하는가.
- 숫자, 날짜, 상대 경로처럼 후처리가 필요한 값을 그대로 두지 않았는가.
- 출력 형태가 문제 요구사항과 일치하는가.

### 자주 보이는 오답

- 이전 문제 selector를 그대로 복사해 현재 구조와 맞지 않는다.
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

CSV를 DictReader로 읽으면 query, category, min_views, max_pages 컬럼을 이름으로 접근할 수 있다. 코드 안에 검색 조건을 고정하지 않고 파일로 분리하는 운영 패턴이다.

### 채점 포인트

- 입력 파일을 환경 셀의 헬퍼로 읽어 코랩과 로컬에서 모두 동작하는가.
- 반복 단위와 selector 또는 URL 처리 함수가 fixture 구조와 정확히 대응하는가.
- 숫자, 날짜, 상대 경로처럼 후처리가 필요한 값을 그대로 두지 않았는가.
- 출력 형태가 문제 요구사항과 일치하는가.

### 자주 보이는 오답

- 이전 문제 selector를 그대로 복사해 현재 구조와 맞지 않는다.
- 문자열 숫자를 변환하지 않고 비교한다.
- 결과는 비슷하지만 중간 변수명이 불분명해 최종 미션에서 재사용하기 어렵다.

---

## 문제 15 정답 — 검색 결과 CSV 저장하기

~~~python
rows = []
for page in range(1, 4):
    page_soup = BeautifulSoup(load_text(page_filename(page)), 'html.parser')
    for card in page_soup.select('article.result-card'):
        rows.append({
            'title': card.select_one('.title a').text.strip(),
            'category': card['data-category'],
            'views': clean_int(card.select_one('.views').text),
        })
with open('lesson02_results.csv', 'w', newline='', encoding='utf-8') as f:
    writer = csv.DictWriter(f, fieldnames=['title', 'category', 'views'])
    writer.writeheader()
    writer.writerows(rows)
print('saved:', 'lesson02_results.csv', len(rows))
~~~

### 왜 이 코드가 정답인지

여러 페이지에서 모은 결과를 같은 키 구조로 만들고 DictWriter로 저장한다. 헤더를 먼저 쓰고 전체 rows를 저장해야 CSV를 다시 열었을 때 컬럼 의미가 유지된다.

### 채점 포인트

- 입력 파일을 환경 셀의 헬퍼로 읽어 코랩과 로컬에서 모두 동작하는가.
- 반복 단위와 selector 또는 URL 처리 함수가 fixture 구조와 정확히 대응하는가.
- 숫자, 날짜, 상대 경로처럼 후처리가 필요한 값을 그대로 두지 않았는가.
- 출력 형태가 문제 요구사항과 일치하는가.

### 자주 보이는 오답

- 이전 문제 selector를 그대로 복사해 현재 구조와 맞지 않는다.
- 문자열 숫자를 변환하지 않고 비교한다.
- 결과는 비슷하지만 중간 변수명이 불분명해 최종 미션에서 재사용하기 어렵다.

---

## 교사용 종합 채점 기준

이 레슨은 문법 암기보다 자동화 흐름을 보는 단원이다. 학생 답안이 출력값을 맞히더라도 URL 조립, selector 기준, 타입 변환, 저장 구조가 불안정하면 부분 감점한다. 반대로 출력 문구가 조금 달라도 중간 데이터 구조가 명확하면 통과로 볼 수 있다.

### URL 처리 기준

문제 1과 2에서는 urlparse, parse_qs, urlencode를 구분해서 쓰는지 확인한다. query string을 split으로만 처리한 답안은 예제에서는 동작할 수 있지만 검색어에 공백이나 특수문자가 들어가면 깨진다. 학생이 왜 표준 함수를 쓰는지 설명할 수 있어야 한다.

### selector 기준

문제 3~5에서는 반복 단위와 내부 selector를 구분하는지 확인한다. article.result-card는 카드 전체를 대표하고, .title a는 카드 내부 제목 링크를 대표한다. 전체 soup에서 매번 첫 링크만 찾는 답안은 반복문으로 확장했을 때 틀리므로 반드시 card 내부에서 select_one을 호출하게 지도한다.

### 데이터 변환 기준

문제 6~8에서는 사람이 보는 문자열을 저장 가능한 값으로 바꾸는지 확인한다. 조회수는 정수, 페이지 번호는 정수, 날짜는 datetime 속성 문자열, 링크는 절대 URL로 정리되어야 한다. 이 변환이 빠지면 이후 필터링이나 CSV 활용이 불안정해진다.

### 페이지 반복 기준

문제 9~11에서는 반복 범위와 중단 기준을 확인한다. range(1, 4)는 1~3페이지를 의미한다. 학생이 range 끝값을 포함한다고 착각하면 3페이지를 빠뜨린다. 다음 링크를 읽는 문제에서는 a.next href를 읽은 뒤 query string에서 page 값을 다시 분해해야 한다.

### 저장 기준

문제 12~15에서는 필터링 기준과 CSV 헤더를 본다. rows에 들어가는 딕셔너리 key와 DictWriter의 fieldnames가 일치해야 한다. 헤더 없이 문자열로 직접 저장하는 답안은 운영자가 다시 읽기 어렵기 때문에 감점한다.

## 수업 중 피드백 문장 예시

- “지금 선택한 단위가 카드 전체인지 카드 안의 링크인지 먼저 말해보자.”
- “이 값은 사람이 읽는 문자열이니, 비교 전에 숫자로 바꿔야 한다.”
- “페이지 반복은 파일명 함수가 맞아야 전체가 맞는다.”
- “CSV는 다음 사람이 다시 여는 파일이라 헤더가 있어야 한다.”

---

## 문제별 추가 확인 질문

문제 1: parse_qs 결과의 값이 리스트인 이유를 설명할 수 있는가? 같은 key가 여러 번 나올 수 있기 때문이다.

문제 2: urlencode를 쓰지 않고 직접 문자열을 붙이면 어떤 검색어에서 깨질 수 있는가? 공백, 한글, 특수문자가 들어간 검색어다.

문제 3: load_text를 쓰는 이유는 무엇인가? 코랩에서는 raw GitHub URL, 로컬에서는 data 폴더를 읽도록 경로 차이를 숨기기 위해서다.

문제 4: article.result-card 개수가 0이면 먼저 무엇을 확인해야 하는가? HTML 파일을 제대로 읽었는지와 class 이름이 맞는지 확인한다.

문제 5~7: title, href, data-category는 각각 텍스트와 속성 중 어디에 있는가? 이 구분을 못 하면 값은 보이는데 저장 구조가 틀어진다.

문제 8~10: 반복문이 페이지별로 새 soup을 만드는지 확인한다. 같은 soup을 계속 쓰면 첫 페이지 결과가 반복된다.

문제 11~15: 저장 전 rows 길이, 첫 행 key, CSV fieldnames를 확인한다. 이 세 가지가 맞으면 대부분의 저장 오류를 미리 잡을 수 있다.

---

## 심화 채점 기준

심화 풀이에서는 새로 HTML을 다시 읽기보다 이미 만든 rows 또는 all_rows를 재사용하는지 확인한다. 데이터를 한 번 구조화한 뒤 필터링과 정렬을 반복 적용하는 습관이 중요하다. 같은 HTML을 불필요하게 여러 번 읽어도 결과는 맞을 수 있지만, 운영 자동화에서는 느리고 관리하기 어렵다.
