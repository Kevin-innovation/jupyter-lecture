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

URL 파라미터와 페이지네이션 레슨의 학생용 문제 노트북이다. 강의 노트북을 먼저 실행한 뒤 빈칸을 직접 채운다. 문제는 URL 구조 이해에서 시작해 여러 페이지 수집과 CSV 저장까지 이어진다.

## 통과 기준

- 총 15문제 중 12문제 이상 정상 출력이면 통과.
- 문제 1~5는 URL과 첫 페이지 구조 확인, 6~10은 링크 정리와 페이지 반복, 11~15는 필터링, 집계, 저장이다.
- 정답값과 완성 코드는 적지 않는다. 출력 형태와 fixture 구조를 보고 직접 판단한다.

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

## 문제 1 — 검색 URL 분해하기

예시 URL에서 path와 query string 값을 분리한다. URL 전체를 문자열로만 보지 않고 검색 조건을 딕셔너리로 읽는 연습이다.

**기대 결과 형태**: path 한 줄과 q/category/page 값 한 줄이 출력된다.

**빈칸 힌트**: urlparse는 URL 구조를 나누고, parse_qs는 query string을 딕셔너리 형태로 바꾼다.

~~~python
sample_url = 'https://example.com/library/search?q=python&category=all&page=2'
parsed = ____(sample_url)
params = ____(parsed.query)
print(parsed.____)
print(params['____'][0], params['____'][0], params['____'][0])
~~~

---

## 문제 2 — 쿼리 문자열 만들기

검색 조건을 딕셔너리로 만든 뒤 URL에 붙인다. 공백이 들어간 검색어가 깨지지 않도록 안전한 조립 방식을 사용한다.

**기대 결과 형태**: 검색어, 카테고리, 페이지 번호가 들어간 URL 문자열이 출력된다.

**빈칸 힌트**: 조건은 딕셔너리로 만들고, query string 변환에는 urlencode를 쓴다.

~~~python
base_url = 'https://example.com/library/search'
query = {'q': ____, 'category': ____, 'page': ____}
url = base_url + '?' + ____(query)
print(url)
~~~

---

## 문제 3 — 첫 페이지 HTML 읽기

search_page_1.html fixture를 읽고 BeautifulSoup 객체로 변환한다. 이후 문제에서 같은 soup 변수를 재사용한다.

**기대 결과 형태**: 첫 페이지의 h1 텍스트가 출력된다.

**빈칸 힌트**: 파일은 load_text로 읽고, HTML parser는 BeautifulSoup(..., html.parser) 형태로 만든다.

~~~python
html_text = ____(____)
soup = ____(html_text, 'html.parser')
print(soup.____('____').text.strip())
~~~

---

## 문제 4 — 결과 카드 개수 세기

검색 결과의 반복 단위인 article.result-card를 모두 선택하고 개수를 확인한다.

**기대 결과 형태**: 카드 개수가 숫자로 출력된다.

**빈칸 힌트**: 반복 단위는 article 태그와 result-card class를 함께 사용한다.

~~~python
cards = soup.____('____')
print('cards:', ____)
~~~

---

## 문제 5 — 첫 카드 제목과 href 읽기

첫 번째 카드에서 제목 텍스트와 링크 경로를 읽는다. 카드 내부 selector를 쓰는 연습이다.

**기대 결과 형태**: 제목 한 줄과 상대 링크 한 줄이 출력된다.

**빈칸 힌트**: 제목 링크는 .title a 위치에 있다. href는 태그 속성으로 읽는다.

~~~python
first = cards[____]
title = first.____('____').text.strip()
href = first.____('____')['____']
print(title)
print(href)
~~~

---

## 문제 6 — 상대 URL을 절대 URL로 바꾸기

카드에서 읽은 상대 링크를 도메인이 포함된 절대 URL로 변환한다.

**기대 결과 형태**: https://example.com으로 시작하는 URL이 출력된다.

**빈칸 힌트**: 기준 도메인과 상대 경로를 합칠 때는 urljoin을 사용한다.

~~~python
absolute = ____('https://example.com', ____)
print(absolute)
~~~

---

## 문제 7 — 카드 하나를 딕셔너리로 만들기

첫 카드에서 제목, 카테고리, 날짜, 조회수, 절대 URL을 뽑아 하나의 딕셔너리로 정리한다.

**기대 결과 형태**: title, category, date, views, url 키를 가진 딕셔너리가 출력된다.

**빈칸 힌트**: 카테고리는 data-category, 날짜는 time의 datetime, 조회수는 clean_int로 정리한다.

~~~python
item = {
    'title': first.select_one('____').text.strip(),
    'category': first['____'],
    'date': first.select_one('____')['____'],
    'views': ____(first.select_one('____').text),
    'url': urljoin('https://example.com', first.select_one('____')['____']),
}
print(item)
~~~

---

## 문제 8 — 한 페이지 결과 리스트 만들기

첫 페이지의 모든 카드를 순회해 딕셔너리 리스트로 만든다.

**기대 결과 형태**: 첫 번째 레코드와 전체 레코드 수가 출력된다.

**빈칸 힌트**: 반복문 안에서 같은 필드를 만들어 results에 추가한다.

~~~python
results = []
for card in cards:
    results.append({
        'title': card.select_one('____').text.strip(),
        'category': card['____'],
        'page': int(card['____']),
        'views': ____(card.select_one('____').text),
    })
print(results[0])
print(len(results))
~~~

---

## 문제 9 — 페이지 번호로 파일명 만들기

페이지 번호를 받아 fixture 파일명을 반환하는 함수를 만든다.

**기대 결과 형태**: 1, 2, 3페이지 파일명이 차례대로 출력된다.

**빈칸 힌트**: 파일명은 search_page_번호.html 규칙이다.

~~~python
def page_filename(page):
    return f'_____{page}.____'

for page in [1, 2, 3]:
    print(____(page))
~~~

---

## 문제 10 — 3페이지 전체 순회하기

1~3페이지 fixture를 순회하면서 모든 카드 제목을 하나의 리스트로 모은다.

**기대 결과 형태**: 전체 제목 개수와 마지막 제목이 출력된다.

**빈칸 힌트**: range(1, 4)는 1, 2, 3을 만든다. 각 페이지마다 page_filename(page)로 파일을 읽는다.

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

pagination 영역의 다음 링크를 읽고 그 안의 page 값을 확인한다.

**기대 결과 형태**: 다음 링크 경로와 다음 페이지 번호가 출력된다.

**빈칸 힌트**: 다음 링크는 a.next이고, page 값은 href의 query string 안에 있다.

~~~python
next_href = soup.select_one('____')['____']
next_query = ____(____(next_href).query)
print(next_href)
print(next_query['____'][0])
~~~

---

## 문제 12 — 조회수 1000 이상 필터링

모든 페이지를 순회하면서 조회수가 1000 이상인 자료 제목만 모은다.

**기대 결과 형태**: 조건을 만족하는 제목 리스트가 출력된다.

**빈칸 힌트**: 조회수는 .views 텍스트를 clean_int로 바꾼 뒤 비교한다.

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

모든 페이지의 카드에서 data-category를 읽어 카테고리별 개수를 집계한다.

**기대 결과 형태**: 카테고리를 키로 가진 딕셔너리가 출력된다.

**빈칸 힌트**: 딕셔너리에 없는 키는 get(key, 0)으로 시작값을 만든다.

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

검색 계획 파일인 search_targets.csv를 읽어 검색어와 최대 페이지 수를 확인한다.

**기대 결과 형태**: 각 행의 query와 max_pages 값이 출력된다.

**빈칸 힌트**: csv.DictReader는 첫 줄 헤더를 딕셔너리 키로 사용한다.

~~~python
target_rows = list(csv.DictReader(load_text('____').splitlines()))
for row in target_rows:
    print(row['____'], row['____'])
~~~

---

## 문제 15 — 검색 결과 CSV 저장하기

모든 페이지의 제목, 카테고리, 조회수를 모아 CSV 파일로 저장한다.

**기대 결과 형태**: 저장 파일명과 저장 행 수가 출력된다.

**빈칸 힌트**: DictWriter는 fieldnames, writeheader, writerows 순서로 사용한다.

~~~python
rows = []
for page in range(1, 4):
    page_soup = BeautifulSoup(load_text(page_filename(page)), 'html.parser')
    for card in page_soup.select('article.result-card'):
        rows.append({
            'title': card.select_one('____').text.strip(),
            'category': card['____'],
            'views': ____(card.select_one('____').text),
        })
with open('lesson02_results.csv', 'w', newline='', encoding='utf-8') as f:
    writer = csv.DictWriter(f, fieldnames=['title', 'category', 'views'])
    writer.____()
    writer.____(rows)
print('saved:', 'lesson02_results.csv', len(rows))
~~~

---

## 풀이 전략

문제 1~5는 첫 페이지를 정확히 읽는 단계다. 여기서 soup, cards, first, href 변수를 제대로 만들어야 뒤 문제가 자연스럽게 이어진다. 변수가 비어 있으면 다음 문제를 억지로 풀지 말고, h1 출력과 카드 개수부터 다시 확인한다.

문제 6~10은 카드 하나를 저장 가능한 데이터로 바꾸고, 같은 작업을 여러 페이지로 확장하는 단계다. 상대 URL, 숫자 변환, page_filename 함수는 이후 문제에서 계속 쓰인다. 함수 이름과 변수 이름을 임의로 바꾸면 뒤 셀을 실행할 때 NameError가 날 수 있다.

문제 11~15는 운영자가 쓰는 형태로 정리하는 단계다. 다음 페이지 링크를 읽고, 조회수 기준으로 필터링하고, 카테고리별로 집계하고, CSV로 저장한다. 출력값만 맞히는 것보다 어떤 기준으로 걸렀는지 설명하는 것이 더 중요하다.

## 제출 전 자체 점검

- 검색 URL을 직접 split하지 않고 urlparse와 parse_qs로 분리했는가?
- 검색 조건 URL을 직접 이어 붙이지 않고 urlencode를 사용했는가?
- 모든 HTML 파일은 load_text로 읽었는가?
- 반복 단위는 article.result-card로 잡았는가?
- 조회수는 clean_int로 정수 변환했는가?
- 상대 링크는 urljoin으로 절대 URL로 바꿨는가?
- 1, 2, 3페이지를 모두 순회했는가?
- CSV 저장 전에 rows 길이와 fieldnames가 맞는지 확인했는가?

## 막혔을 때 확인할 값

첫 번째로 page_filename(1)의 결과를 확인한다. 두 번째로 search_page_1.html의 h1을 출력한다. 세 번째로 len(cards)를 출력한다. 네 번째로 cards[0]에서 title과 href가 나오는지 확인한다. 이 네 값이 맞으면 대부분의 문제는 selector나 타입 변환 실수다.

---

## 문제별 변수 연결표

| 구간 | 핵심 변수 | 뒤에서 쓰이는 곳 |
|---|---|---|
| 문제 1~2 | parsed, params, url | URL 구조 설명과 검색 조건 조립 |
| 문제 3~5 | soup, cards, first, href | 첫 페이지 selector 확인 |
| 문제 6~8 | absolute, item, results | 저장 가능한 레코드 구조 만들기 |
| 문제 9~10 | page_filename, all_results | 여러 페이지 반복 |
| 문제 11~15 | next_query, rich, counts, target_rows, rows | 운영 요약과 CSV 저장 |

변수는 정답을 맞히기 위한 임시 이름이 아니라 다음 셀과 연결되는 약속이다. 이름을 바꿔도 되지만, 바꾼 뒤에는 뒤 셀에서도 같은 이름으로 맞춰야 한다. 수업 중에는 NameError가 나면 바로 정답을 보지 말고 어느 문제에서 변수가 만들어졌는지 먼저 거슬러 올라간다.

## 힌트 사용 기준

힌트는 완성 코드를 알려주는 용도가 아니다. 어떤 HTML 구조를 봐야 하는지, 어떤 표준 함수를 떠올려야 하는지 알려주는 방향으로만 사용한다. 빈칸을 채울 때는 바로 실행하지 말고, 그 빈칸이 함수 이름인지 selector인지 속성 이름인지 먼저 구분한다.

---

## 최소 통과 후 심화 방향

12문제 이상 통과한 학생은 CSV 저장 후 category가 python인 행만 다시 골라본다. 그다음 views 기준으로 내림차순 정렬해 상위 3개 제목을 출력한다. 이 심화는 새 문법을 요구하지 않고, 이미 만든 rows 구조를 재사용하는 연습이다.
