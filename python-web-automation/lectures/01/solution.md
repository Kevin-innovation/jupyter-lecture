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

# 레슨 01 — 실습 문제 정답지

> 🔒 교사·관리자 전용. 학생에게 배포 금지.

각 문제의 모범 답안과 채점 포인트를 정리했다. 학생 답안은 출력값만 보지 말고 selector가 안정적인지, 텍스트 정리와 타입 변환이 정확한지 확인한다.

## 0. 환경 셀

~~~python
import os
import re
import time
import csv
from pathlib import Path
from urllib.parse import urljoin

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
    DATA_BASE = 'https://raw.githubusercontent.com/Kevin-innovation/jupyter-lecture/main/python-web-automation/lectures/01/data'
else:
    DATA_BASE = './data'

def load_text(filename):
    if DATA_BASE.startswith('http'):
        url = f'{DATA_BASE}/{filename}'
        response = requests.get(url, timeout=10)
        response.raise_for_status()
        response.encoding = response.encoding or 'utf-8'
        return response.text
    return Path(DATA_BASE, filename).read_text(encoding='utf-8')

def clean_int(text):
    return int(re.sub(r'[^0-9]', '', text))

print('colab:', IS_COLAB)
print('data base:', DATA_BASE)
~~~

---

## 문제 1 정답 — 환경 셀 실행과 파일 목록 확인

~~~python
shop_html = load_text('mini_shop.html')
notice_html = load_text('notices.html')
target_csv = load_text('targets.csv')
print('shop:', len(shop_html))
print('notice:', len(notice_html))
print('targets:', len(target_csv))
~~~

### 왜 이 코드가 정답인지

이 코드는 문제에서 요구한 HTML 또는 CSV 원본을 먼저 올바른 자료구조로 바꾼 뒤, 필요한 위치만 선택한다. BeautifulSoup을 사용할 때는 태그 이름보다 class 기반 selector를 쓰는 편이 수업용 HTML 구조와 잘 맞고, 텍스트는 strip으로 공백을 제거한다. 가격·재고·조회수처럼 비교나 정렬이 필요한 값은 clean_int로 숫자만 남겨 int로 바꾼다. 이 변환이 빠지면 문자열 정렬이나 문자열 비교가 되어 잘못된 결과가 나온다.

### 채점 포인트

- selector가 실제 HTML 구조와 맞는가.
- text와 attribute를 구분해서 읽었는가.
- 숫자 필드는 int 또는 float로 변환했는가.
- 반복 추출 문제는 리스트 길이가 원본 카드/행 개수와 일치하는가.

### 자주 보이는 오답

- select_one 결과에 바로 인덱스를 붙이거나, select 결과 리스트에서 text를 바로 읽으려는 실수.
- 가격 문자열을 숫자로 바꾸지 않고 그대로 비교하는 실수.
- href를 상대 경로 그대로 저장해 나중에 열 수 없는 링크가 되는 실수.

---

## 문제 2 정답 — HTML 제목과 h1 찾기

~~~python
soup = BeautifulSoup(shop_html, 'html.parser')
page_title = soup.title.text.strip()
main_title = soup.select_one('h1').text.strip()
print(page_title)
print(main_title)
~~~

### 왜 이 코드가 정답인지

이 코드는 문제에서 요구한 HTML 또는 CSV 원본을 먼저 올바른 자료구조로 바꾼 뒤, 필요한 위치만 선택한다. BeautifulSoup을 사용할 때는 태그 이름보다 class 기반 selector를 쓰는 편이 수업용 HTML 구조와 잘 맞고, 텍스트는 strip으로 공백을 제거한다. 가격·재고·조회수처럼 비교나 정렬이 필요한 값은 clean_int로 숫자만 남겨 int로 바꾼다. 이 변환이 빠지면 문자열 정렬이나 문자열 비교가 되어 잘못된 결과가 나온다.

### 채점 포인트

- selector가 실제 HTML 구조와 맞는가.
- text와 attribute를 구분해서 읽었는가.
- 숫자 필드는 int 또는 float로 변환했는가.
- 반복 추출 문제는 리스트 길이가 원본 카드/행 개수와 일치하는가.

### 자주 보이는 오답

- select_one 결과에 바로 인덱스를 붙이거나, select 결과 리스트에서 text를 바로 읽으려는 실수.
- 가격 문자열을 숫자로 바꾸지 않고 그대로 비교하는 실수.
- href를 상대 경로 그대로 저장해 나중에 열 수 없는 링크가 되는 실수.

---

## 문제 3 정답 — 상품 카드 개수 세기

~~~python
cards = soup.select('.product-card')
print('상품 카드 수:', len(cards))
~~~

### 왜 이 코드가 정답인지

이 코드는 문제에서 요구한 HTML 또는 CSV 원본을 먼저 올바른 자료구조로 바꾼 뒤, 필요한 위치만 선택한다. BeautifulSoup을 사용할 때는 태그 이름보다 class 기반 selector를 쓰는 편이 수업용 HTML 구조와 잘 맞고, 텍스트는 strip으로 공백을 제거한다. 가격·재고·조회수처럼 비교나 정렬이 필요한 값은 clean_int로 숫자만 남겨 int로 바꾼다. 이 변환이 빠지면 문자열 정렬이나 문자열 비교가 되어 잘못된 결과가 나온다.

### 채점 포인트

- selector가 실제 HTML 구조와 맞는가.
- text와 attribute를 구분해서 읽었는가.
- 숫자 필드는 int 또는 float로 변환했는가.
- 반복 추출 문제는 리스트 길이가 원본 카드/행 개수와 일치하는가.

### 자주 보이는 오답

- select_one 결과에 바로 인덱스를 붙이거나, select 결과 리스트에서 text를 바로 읽으려는 실수.
- 가격 문자열을 숫자로 바꾸지 않고 그대로 비교하는 실수.
- href를 상대 경로 그대로 저장해 나중에 열 수 없는 링크가 되는 실수.

---

## 문제 4 정답 — 첫 번째 상품 정보 읽기

~~~python
first = cards[0]
name = first.select_one('.name').text.strip()
category = first['data-category']
price_text = first.select_one('.price').text.strip()
href = first.select_one('a.detail')['href']
print(name, category, price_text, href)
~~~

### 왜 이 코드가 정답인지

이 코드는 문제에서 요구한 HTML 또는 CSV 원본을 먼저 올바른 자료구조로 바꾼 뒤, 필요한 위치만 선택한다. BeautifulSoup을 사용할 때는 태그 이름보다 class 기반 selector를 쓰는 편이 수업용 HTML 구조와 잘 맞고, 텍스트는 strip으로 공백을 제거한다. 가격·재고·조회수처럼 비교나 정렬이 필요한 값은 clean_int로 숫자만 남겨 int로 바꾼다. 이 변환이 빠지면 문자열 정렬이나 문자열 비교가 되어 잘못된 결과가 나온다.

### 채점 포인트

- selector가 실제 HTML 구조와 맞는가.
- text와 attribute를 구분해서 읽었는가.
- 숫자 필드는 int 또는 float로 변환했는가.
- 반복 추출 문제는 리스트 길이가 원본 카드/행 개수와 일치하는가.

### 자주 보이는 오답

- select_one 결과에 바로 인덱스를 붙이거나, select 결과 리스트에서 text를 바로 읽으려는 실수.
- 가격 문자열을 숫자로 바꾸지 않고 그대로 비교하는 실수.
- href를 상대 경로 그대로 저장해 나중에 열 수 없는 링크가 되는 실수.

---

## 문제 5 정답 — 가격 문자열을 숫자로 바꾸기

~~~python
price_text = first.select_one('.price').text
price = clean_int(price_text)
print(price, type(price))
~~~

### 왜 이 코드가 정답인지

이 코드는 문제에서 요구한 HTML 또는 CSV 원본을 먼저 올바른 자료구조로 바꾼 뒤, 필요한 위치만 선택한다. BeautifulSoup을 사용할 때는 태그 이름보다 class 기반 selector를 쓰는 편이 수업용 HTML 구조와 잘 맞고, 텍스트는 strip으로 공백을 제거한다. 가격·재고·조회수처럼 비교나 정렬이 필요한 값은 clean_int로 숫자만 남겨 int로 바꾼다. 이 변환이 빠지면 문자열 정렬이나 문자열 비교가 되어 잘못된 결과가 나온다.

### 채점 포인트

- selector가 실제 HTML 구조와 맞는가.
- text와 attribute를 구분해서 읽었는가.
- 숫자 필드는 int 또는 float로 변환했는가.
- 반복 추출 문제는 리스트 길이가 원본 카드/행 개수와 일치하는가.

### 자주 보이는 오답

- select_one 결과에 바로 인덱스를 붙이거나, select 결과 리스트에서 text를 바로 읽으려는 실수.
- 가격 문자열을 숫자로 바꾸지 않고 그대로 비교하는 실수.
- href를 상대 경로 그대로 저장해 나중에 열 수 없는 링크가 되는 실수.

---

## 문제 6 정답 — 모든 상품을 딕셔너리 리스트로 만들기

~~~python
products = []
for card in cards:
    item = {
        'name': card.select_one('.name').text.strip(),
        'category': card['data-category'],
        'price': clean_int(card.select_one('.price').text),
        'rating': float(card.select_one('.rating').text.replace('★', '').strip()),
        'stock': clean_int(card.select_one('.stock').text),
        'detail_url': urljoin('https://example.com', card.select_one('a.detail')['href']),
    }
    products.append(item)
print(products[0])
print(len(products))
~~~

### 왜 이 코드가 정답인지

이 코드는 문제에서 요구한 HTML 또는 CSV 원본을 먼저 올바른 자료구조로 바꾼 뒤, 필요한 위치만 선택한다. BeautifulSoup을 사용할 때는 태그 이름보다 class 기반 selector를 쓰는 편이 수업용 HTML 구조와 잘 맞고, 텍스트는 strip으로 공백을 제거한다. 가격·재고·조회수처럼 비교나 정렬이 필요한 값은 clean_int로 숫자만 남겨 int로 바꾼다. 이 변환이 빠지면 문자열 정렬이나 문자열 비교가 되어 잘못된 결과가 나온다.

### 채점 포인트

- selector가 실제 HTML 구조와 맞는가.
- text와 attribute를 구분해서 읽었는가.
- 숫자 필드는 int 또는 float로 변환했는가.
- 반복 추출 문제는 리스트 길이가 원본 카드/행 개수와 일치하는가.

### 자주 보이는 오답

- select_one 결과에 바로 인덱스를 붙이거나, select 결과 리스트에서 text를 바로 읽으려는 실수.
- 가격 문자열을 숫자로 바꾸지 않고 그대로 비교하는 실수.
- href를 상대 경로 그대로 저장해 나중에 열 수 없는 링크가 되는 실수.

---

## 문제 7 정답 — 카테고리별 상품 수 세기

~~~python
counts = {}
for item in products:
    key = item['category']
    counts[key] = counts.get(key, 0) + 1
print(counts)
~~~

### 왜 이 코드가 정답인지

이 코드는 문제에서 요구한 HTML 또는 CSV 원본을 먼저 올바른 자료구조로 바꾼 뒤, 필요한 위치만 선택한다. BeautifulSoup을 사용할 때는 태그 이름보다 class 기반 selector를 쓰는 편이 수업용 HTML 구조와 잘 맞고, 텍스트는 strip으로 공백을 제거한다. 가격·재고·조회수처럼 비교나 정렬이 필요한 값은 clean_int로 숫자만 남겨 int로 바꾼다. 이 변환이 빠지면 문자열 정렬이나 문자열 비교가 되어 잘못된 결과가 나온다.

### 채점 포인트

- selector가 실제 HTML 구조와 맞는가.
- text와 attribute를 구분해서 읽었는가.
- 숫자 필드는 int 또는 float로 변환했는가.
- 반복 추출 문제는 리스트 길이가 원본 카드/행 개수와 일치하는가.

### 자주 보이는 오답

- select_one 결과에 바로 인덱스를 붙이거나, select 결과 리스트에서 text를 바로 읽으려는 실수.
- 가격 문자열을 숫자로 바꾸지 않고 그대로 비교하는 실수.
- href를 상대 경로 그대로 저장해 나중에 열 수 없는 링크가 되는 실수.

---

## 문제 8 정답 — 150만원 이상 상품 찾기

~~~python
expensive = [p for p in products if p['price'] >= 1500000]
for p in expensive:
    print(p['name'], p['price'])
~~~

### 왜 이 코드가 정답인지

이 코드는 문제에서 요구한 HTML 또는 CSV 원본을 먼저 올바른 자료구조로 바꾼 뒤, 필요한 위치만 선택한다. BeautifulSoup을 사용할 때는 태그 이름보다 class 기반 selector를 쓰는 편이 수업용 HTML 구조와 잘 맞고, 텍스트는 strip으로 공백을 제거한다. 가격·재고·조회수처럼 비교나 정렬이 필요한 값은 clean_int로 숫자만 남겨 int로 바꾼다. 이 변환이 빠지면 문자열 정렬이나 문자열 비교가 되어 잘못된 결과가 나온다.

### 채점 포인트

- selector가 실제 HTML 구조와 맞는가.
- text와 attribute를 구분해서 읽었는가.
- 숫자 필드는 int 또는 float로 변환했는가.
- 반복 추출 문제는 리스트 길이가 원본 카드/행 개수와 일치하는가.

### 자주 보이는 오답

- select_one 결과에 바로 인덱스를 붙이거나, select 결과 리스트에서 text를 바로 읽으려는 실수.
- 가격 문자열을 숫자로 바꾸지 않고 그대로 비교하는 실수.
- href를 상대 경로 그대로 저장해 나중에 열 수 없는 링크가 되는 실수.

---

## 문제 9 정답 — 평점 4.7 이상 상품 찾기

~~~python
top_rated = [p for p in products if p['rating'] >= 4.7]
for p in top_rated:
    print(p['name'], p['rating'], p['stock'])
~~~

### 왜 이 코드가 정답인지

이 코드는 문제에서 요구한 HTML 또는 CSV 원본을 먼저 올바른 자료구조로 바꾼 뒤, 필요한 위치만 선택한다. BeautifulSoup을 사용할 때는 태그 이름보다 class 기반 selector를 쓰는 편이 수업용 HTML 구조와 잘 맞고, 텍스트는 strip으로 공백을 제거한다. 가격·재고·조회수처럼 비교나 정렬이 필요한 값은 clean_int로 숫자만 남겨 int로 바꾼다. 이 변환이 빠지면 문자열 정렬이나 문자열 비교가 되어 잘못된 결과가 나온다.

### 채점 포인트

- selector가 실제 HTML 구조와 맞는가.
- text와 attribute를 구분해서 읽었는가.
- 숫자 필드는 int 또는 float로 변환했는가.
- 반복 추출 문제는 리스트 길이가 원본 카드/행 개수와 일치하는가.

### 자주 보이는 오답

- select_one 결과에 바로 인덱스를 붙이거나, select 결과 리스트에서 text를 바로 읽으려는 실수.
- 가격 문자열을 숫자로 바꾸지 않고 그대로 비교하는 실수.
- href를 상대 경로 그대로 저장해 나중에 열 수 없는 링크가 되는 실수.

---

## 문제 10 정답 — 품절 상품 찾기

~~~python
sold_out = [p for p in products if p['stock'] == 0]
for p in sold_out:
    print(p['name'], p['detail_url'])
~~~

### 왜 이 코드가 정답인지

이 코드는 문제에서 요구한 HTML 또는 CSV 원본을 먼저 올바른 자료구조로 바꾼 뒤, 필요한 위치만 선택한다. BeautifulSoup을 사용할 때는 태그 이름보다 class 기반 selector를 쓰는 편이 수업용 HTML 구조와 잘 맞고, 텍스트는 strip으로 공백을 제거한다. 가격·재고·조회수처럼 비교나 정렬이 필요한 값은 clean_int로 숫자만 남겨 int로 바꾼다. 이 변환이 빠지면 문자열 정렬이나 문자열 비교가 되어 잘못된 결과가 나온다.

### 채점 포인트

- selector가 실제 HTML 구조와 맞는가.
- text와 attribute를 구분해서 읽었는가.
- 숫자 필드는 int 또는 float로 변환했는가.
- 반복 추출 문제는 리스트 길이가 원본 카드/행 개수와 일치하는가.

### 자주 보이는 오답

- select_one 결과에 바로 인덱스를 붙이거나, select 결과 리스트에서 text를 바로 읽으려는 실수.
- 가격 문자열을 숫자로 바꾸지 않고 그대로 비교하는 실수.
- href를 상대 경로 그대로 저장해 나중에 열 수 없는 링크가 되는 실수.

---

## 문제 11 정답 — 공지 목록 파싱하기

~~~python
notice_soup = BeautifulSoup(notice_html, 'html.parser')
notice_rows = notice_soup.select('li.notice-item')
notices = []
for row in notice_rows:
    notices.append({
        'date': row.select_one('.date').text.strip(),
        'title': row.select_one('.title').text.strip(),
        'department': row.select_one('.department').text.strip(),
        'views': clean_int(row.select_one('.views').text),
    })
print(notices[0])
print(len(notices))
~~~

### 왜 이 코드가 정답인지

이 코드는 문제에서 요구한 HTML 또는 CSV 원본을 먼저 올바른 자료구조로 바꾼 뒤, 필요한 위치만 선택한다. BeautifulSoup을 사용할 때는 태그 이름보다 class 기반 selector를 쓰는 편이 수업용 HTML 구조와 잘 맞고, 텍스트는 strip으로 공백을 제거한다. 가격·재고·조회수처럼 비교나 정렬이 필요한 값은 clean_int로 숫자만 남겨 int로 바꾼다. 이 변환이 빠지면 문자열 정렬이나 문자열 비교가 되어 잘못된 결과가 나온다.

### 채점 포인트

- selector가 실제 HTML 구조와 맞는가.
- text와 attribute를 구분해서 읽었는가.
- 숫자 필드는 int 또는 float로 변환했는가.
- 반복 추출 문제는 리스트 길이가 원본 카드/행 개수와 일치하는가.

### 자주 보이는 오답

- select_one 결과에 바로 인덱스를 붙이거나, select 결과 리스트에서 text를 바로 읽으려는 실수.
- 가격 문자열을 숫자로 바꾸지 않고 그대로 비교하는 실수.
- href를 상대 경로 그대로 저장해 나중에 열 수 없는 링크가 되는 실수.

---

## 문제 12 정답 — 운영팀 공지만 필터링하기

~~~python
ops_notices = [n for n in notices if n['department'] == '운영팀']
for n in ops_notices:
    print(n['date'], n['title'])
~~~

### 왜 이 코드가 정답인지

이 코드는 문제에서 요구한 HTML 또는 CSV 원본을 먼저 올바른 자료구조로 바꾼 뒤, 필요한 위치만 선택한다. BeautifulSoup을 사용할 때는 태그 이름보다 class 기반 selector를 쓰는 편이 수업용 HTML 구조와 잘 맞고, 텍스트는 strip으로 공백을 제거한다. 가격·재고·조회수처럼 비교나 정렬이 필요한 값은 clean_int로 숫자만 남겨 int로 바꾼다. 이 변환이 빠지면 문자열 정렬이나 문자열 비교가 되어 잘못된 결과가 나온다.

### 채점 포인트

- selector가 실제 HTML 구조와 맞는가.
- text와 attribute를 구분해서 읽었는가.
- 숫자 필드는 int 또는 float로 변환했는가.
- 반복 추출 문제는 리스트 길이가 원본 카드/행 개수와 일치하는가.

### 자주 보이는 오답

- select_one 결과에 바로 인덱스를 붙이거나, select 결과 리스트에서 text를 바로 읽으려는 실수.
- 가격 문자열을 숫자로 바꾸지 않고 그대로 비교하는 실수.
- href를 상대 경로 그대로 저장해 나중에 열 수 없는 링크가 되는 실수.

---

## 문제 13 정답 — 조회수 상위 공지 3개 찾기

~~~python
top3 = sorted(notices, key=lambda n: n['views'], reverse=True)[:3]
for n in top3:
    print(n['title'], n['views'])
~~~

### 왜 이 코드가 정답인지

이 코드는 문제에서 요구한 HTML 또는 CSV 원본을 먼저 올바른 자료구조로 바꾼 뒤, 필요한 위치만 선택한다. BeautifulSoup을 사용할 때는 태그 이름보다 class 기반 selector를 쓰는 편이 수업용 HTML 구조와 잘 맞고, 텍스트는 strip으로 공백을 제거한다. 가격·재고·조회수처럼 비교나 정렬이 필요한 값은 clean_int로 숫자만 남겨 int로 바꾼다. 이 변환이 빠지면 문자열 정렬이나 문자열 비교가 되어 잘못된 결과가 나온다.

### 채점 포인트

- selector가 실제 HTML 구조와 맞는가.
- text와 attribute를 구분해서 읽었는가.
- 숫자 필드는 int 또는 float로 변환했는가.
- 반복 추출 문제는 리스트 길이가 원본 카드/행 개수와 일치하는가.

### 자주 보이는 오답

- select_one 결과에 바로 인덱스를 붙이거나, select 결과 리스트에서 text를 바로 읽으려는 실수.
- 가격 문자열을 숫자로 바꾸지 않고 그대로 비교하는 실수.
- href를 상대 경로 그대로 저장해 나중에 열 수 없는 링크가 되는 실수.

---

## 문제 14 정답 — targets.csv에서 허용 대상만 읽기

~~~python
import io
reader = csv.DictReader(io.StringIO(target_csv))
allowed = []
for row in reader:
    if row['allowed'] == 'yes':
        allowed.append(row)
for row in allowed:
    print(row['filename'], row['purpose'])
~~~

### 왜 이 코드가 정답인지

이 코드는 문제에서 요구한 HTML 또는 CSV 원본을 먼저 올바른 자료구조로 바꾼 뒤, 필요한 위치만 선택한다. BeautifulSoup을 사용할 때는 태그 이름보다 class 기반 selector를 쓰는 편이 수업용 HTML 구조와 잘 맞고, 텍스트는 strip으로 공백을 제거한다. 가격·재고·조회수처럼 비교나 정렬이 필요한 값은 clean_int로 숫자만 남겨 int로 바꾼다. 이 변환이 빠지면 문자열 정렬이나 문자열 비교가 되어 잘못된 결과가 나온다.

### 채점 포인트

- selector가 실제 HTML 구조와 맞는가.
- text와 attribute를 구분해서 읽었는가.
- 숫자 필드는 int 또는 float로 변환했는가.
- 반복 추출 문제는 리스트 길이가 원본 카드/행 개수와 일치하는가.

### 자주 보이는 오답

- select_one 결과에 바로 인덱스를 붙이거나, select 결과 리스트에서 text를 바로 읽으려는 실수.
- 가격 문자열을 숫자로 바꾸지 않고 그대로 비교하는 실수.
- href를 상대 경로 그대로 저장해 나중에 열 수 없는 링크가 되는 실수.

---

## 문제 15 정답 — 수집 요약 문장 만들기

~~~python
summary1 = f"총 상품 수는 {len(products)}개입니다."
summary2 = f"품절 상품은 {len(sold_out)}개입니다."
summary3 = f"조회수 1위 공지는 '{top3[0]['title']}'입니다."
print(summary1)
print(summary2)
print(summary3)
~~~

### 왜 이 코드가 정답인지

이 코드는 문제에서 요구한 HTML 또는 CSV 원본을 먼저 올바른 자료구조로 바꾼 뒤, 필요한 위치만 선택한다. BeautifulSoup을 사용할 때는 태그 이름보다 class 기반 selector를 쓰는 편이 수업용 HTML 구조와 잘 맞고, 텍스트는 strip으로 공백을 제거한다. 가격·재고·조회수처럼 비교나 정렬이 필요한 값은 clean_int로 숫자만 남겨 int로 바꾼다. 이 변환이 빠지면 문자열 정렬이나 문자열 비교가 되어 잘못된 결과가 나온다.

### 채점 포인트

- selector가 실제 HTML 구조와 맞는가.
- text와 attribute를 구분해서 읽었는가.
- 숫자 필드는 int 또는 float로 변환했는가.
- 반복 추출 문제는 리스트 길이가 원본 카드/행 개수와 일치하는가.

### 자주 보이는 오답

- select_one 결과에 바로 인덱스를 붙이거나, select 결과 리스트에서 text를 바로 읽으려는 실수.
- 가격 문자열을 숫자로 바꾸지 않고 그대로 비교하는 실수.
- href를 상대 경로 그대로 저장해 나중에 열 수 없는 링크가 되는 실수.

---

## 채점 시 우선 확인 순서

채점은 문제 번호 순서대로 하되, 실제로는 의존 관계를 먼저 본다. 문제 1~3에서 환경과 반복 단위가 맞지 않으면 뒤 문제 대부분이 연쇄적으로 실패한다. 이 경우 뒤 문제를 하나씩 고치게 하기보다 `shop_html`, `soup`, `cards` 가 만들어지는 흐름을 먼저 복구하게 한다.

| 확인 순서 | 봐야 할 것 | 통과 기준 |
|---|---|---|
| 1 | 환경 셀 | `DATA_BASE`, `load_text`, `clean_int` 가 정상 정의됨 |
| 2 | HTML 로드 | `shop_html`, `notice_html`, `target_csv` 가 비어 있지 않음 |
| 3 | 반복 단위 | `.product-card`, `li.notice-item` 개수가 원본과 일치 |
| 4 | 타입 변환 | 가격/재고/조회수가 숫자로 변환됨 |
| 5 | 저장 | `lesson01_products.csv` 가 헤더와 행을 포함 |
| 6 | 요약 | 마지막 문장이 실제 출력 숫자와 충돌하지 않음 |

## 부분 점수 안내

학생이 모범 답안과 다른 코드를 작성해도 같은 구조를 만족하면 통과시킬 수 있다. 예를 들어 카테고리별 집계를 `collections.Counter` 로 풀어도 좋고, CSV 읽기를 `pandas.read_csv` 로 시도한 학생도 결과가 맞으면 인정할 수 있다. 다만 1강의 학습 목표가 BeautifulSoup selector와 표준 라이브러리 `csv` 흐름이므로, 다른 도구로 풀었더라도 수업 기준 풀이를 한 번 더 설명하게 한다.

감점 또는 보완 대상은 다음과 같다.

- selector 결과가 `None` 인데 `.text` 를 바로 호출해 노트북이 중간에 멈춘다.
- 가격을 문자열 상태로 비교해 `"900000"` 과 `"1500000"` 의 순서가 잘못된다.
- 상대 URL을 그대로 저장해 CSV만 봤을 때 링크의 기준 사이트를 알 수 없다.
- 실제 사이트에 바로 적용하겠다고 쓰면서 robots.txt, 약관, 요청 간격 언급이 없다.
- 결과 요약 문장이 코드 출력과 맞지 않는다.
