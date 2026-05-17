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

# 레슨 01 — 실습 문제

lecture.ipynb를 본 뒤 진행하는 학생용 문제 노트북이다. 이번 레슨은 정적 HTML 파일을 읽고, BeautifulSoup selector로 필요한 값을 뽑아 표 형태로 정리하는 것이 목표다.

## 통과 기준

- 총 15문제 중 12문제 이상 정상 출력이면 통과.
- 문제 1~5: 환경, HTML 기본 구조, 단일 요소 추출.
- 문제 6~10: 상품 카드 반복 추출과 조건 필터.
- 문제 11~14: 공지/대상 CSV 파싱.
- 문제 15: 수집 결과를 근거로 한 문장 요약 작성.

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

## 문제 1 — 환경 셀 실행과 파일 목록 확인

mini_shop.html, notices.html, targets.csv를 load_text로 읽고 각 문자열 길이를 출력한다.

**기대 결과 형태**: 파일명과 글자 수가 3줄로 나온다.

**빈칸 힌트**: 코드 셀의 빈칸을 채운다. selector 문자열과 딕셔너리 key는 lecture 노트북의 예제와 HTML 구조를 같이 보고 결정한다.

~~~python
# TODO: ____ 부분을 직접 채우세요.
shop_html = ____(____)
notice_html = ____(____)
target_csv = ____(____)
print('shop:', ____)
print('notice:', ____)
print('targets:', ____)
~~~

---

## 문제 2 — HTML 제목과 h1 찾기

mini_shop.html에서 title과 h1 텍스트를 출력한다.

**기대 결과 형태**: 문서 제목 1줄, 화면 제목 1줄이 나온다.

**빈칸 힌트**: 코드 셀의 빈칸을 채운다. selector 문자열과 딕셔너리 key는 lecture 노트북의 예제와 HTML 구조를 같이 보고 결정한다.

~~~python
# TODO: ____ 부분을 직접 채우세요.
soup = ____(shop_html, 'html.parser')
page_title = soup.____.text.strip()
main_title = soup.____('____').text.strip()
print(page_title)
print(main_title)
~~~

---

## 문제 3 — 상품 카드 개수 세기

class가 product-card인 요소를 모두 선택해 개수를 출력한다.

**기대 결과 형태**: 상품 카드 수가 한 줄로 나온다.

**빈칸 힌트**: 코드 셀의 빈칸을 채운다. selector 문자열과 딕셔너리 key는 lecture 노트북의 예제와 HTML 구조를 같이 보고 결정한다.

~~~python
# TODO: ____ 부분을 직접 채우세요.
cards = soup.____('____')
print('상품 카드 수:', ____)
~~~

---

## 문제 4 — 첫 번째 상품 정보 읽기

첫 상품의 이름, 카테고리, 가격 문자열, 상세 링크를 출력한다.

**기대 결과 형태**: 이름/category/가격/href가 각각 나온다.

**빈칸 힌트**: 코드 셀의 빈칸을 채운다. selector 문자열과 딕셔너리 key는 lecture 노트북의 예제와 HTML 구조를 같이 보고 결정한다.

~~~python
# TODO: ____ 부분을 직접 채우세요.
first = cards[____]
name = first.____('____').text.strip()
category = first['____']
price_text = first.____('____').text.strip()
href = first.____('____')['____']
print(name, category, price_text, href)
~~~

---

## 문제 5 — 가격 문자열을 숫자로 바꾸기

첫 상품 가격에서 숫자만 남겨 int로 변환한다.

**기대 결과 형태**: 정수 가격이 출력된다.

**빈칸 힌트**: 코드 셀의 빈칸을 채운다. selector 문자열과 딕셔너리 key는 lecture 노트북의 예제와 HTML 구조를 같이 보고 결정한다.

~~~python
# TODO: ____ 부분을 직접 채우세요.
price_text = first.select_one('.price').text
price = ____(price_text)
print(price, type(price))
~~~

---

## 문제 6 — 모든 상품을 딕셔너리 리스트로 만들기

각 상품에서 name/category/price/rating/stock/detail_url을 추출해 products 리스트를 만든다.

**기대 결과 형태**: 첫 딕셔너리와 전체 개수가 출력된다.

**빈칸 힌트**: 코드 셀의 빈칸을 채운다. selector 문자열과 딕셔너리 key는 lecture 노트북의 예제와 HTML 구조를 같이 보고 결정한다.

~~~python
# TODO: ____ 부분을 직접 채우세요.
products = []
for card in cards:
    item = {
        'name': card.select_one('____').text.strip(),
        'category': card['____'],
        'price': ____(card.select_one('____').text),
        'rating': float(card.select_one('____').text.replace('★', '').strip()),
        'stock': ____(card.select_one('____').text),
        'detail_url': urljoin('https://example.com', card.select_one('____')['____']),
    }
    products.append(item)
print(products[0])
print(len(products))
~~~

---

## 문제 7 — 카테고리별 상품 수 세기

products에서 category별 상품 수를 딕셔너리로 집계한다.

**기대 결과 형태**: 카테고리 이름과 개수가 출력된다.

**빈칸 힌트**: 코드 셀의 빈칸을 채운다. selector 문자열과 딕셔너리 key는 lecture 노트북의 예제와 HTML 구조를 같이 보고 결정한다.

~~~python
# TODO: ____ 부분을 직접 채우세요.
counts = {}
for item in products:
    key = item['____']
    counts[key] = counts.get(key, 0) + ____
print(counts)
~~~

---

## 문제 8 — 150만원 이상 상품 찾기

가격이 1,500,000원 이상인 상품명과 가격을 출력한다.

**기대 결과 형태**: 조건을 만족하는 상품 목록이 나온다.

**빈칸 힌트**: 코드 셀의 빈칸을 채운다. selector 문자열과 딕셔너리 key는 lecture 노트북의 예제와 HTML 구조를 같이 보고 결정한다.

~~~python
# TODO: ____ 부분을 직접 채우세요.
expensive = [p for p in products if p['____'] >= ____]
for p in expensive:
    print(p['____'], p['____'])
~~~

---

## 문제 9 — 평점 4.7 이상 상품 찾기

rating이 4.7 이상인 상품만 골라 이름, 평점, 재고를 출력한다.

**기대 결과 형태**: 고평점 상품 목록이 나온다.

**빈칸 힌트**: 코드 셀의 빈칸을 채운다. selector 문자열과 딕셔너리 key는 lecture 노트북의 예제와 HTML 구조를 같이 보고 결정한다.

~~~python
# TODO: ____ 부분을 직접 채우세요.
top_rated = [p for p in products if p['____'] >= ____]
for p in top_rated:
    print(p['____'], p['____'], p['____'])
~~~

---

## 문제 10 — 품절 상품 찾기

stock이 0인 상품의 이름과 상세 URL을 출력한다.

**기대 결과 형태**: 품절 상품 목록이 나온다.

**빈칸 힌트**: 코드 셀의 빈칸을 채운다. selector 문자열과 딕셔너리 key는 lecture 노트북의 예제와 HTML 구조를 같이 보고 결정한다.

~~~python
# TODO: ____ 부분을 직접 채우세요.
sold_out = [p for p in products if p['____'] == ____]
for p in sold_out:
    print(p['____'], p['____'])
~~~

---

## 문제 11 — 공지 목록 파싱하기

notices.html에서 notice-item을 모두 골라 날짜, 제목, 부서를 추출한다.

**기대 결과 형태**: 첫 공지 딕셔너리와 전체 공지 수가 출력된다.

**빈칸 힌트**: 코드 셀의 빈칸을 채운다. selector 문자열과 딕셔너리 key는 lecture 노트북의 예제와 HTML 구조를 같이 보고 결정한다.

~~~python
# TODO: ____ 부분을 직접 채우세요.
notice_soup = ____(notice_html, 'html.parser')
notice_rows = notice_soup.____('____')
notices = []
for row in notice_rows:
    notices.append({
        'date': row.select_one('____').text.strip(),
        'title': row.select_one('____').text.strip(),
        'department': row.select_one('____').text.strip(),
        'views': ____(row.select_one('____').text),
    })
print(notices[0])
print(len(notices))
~~~

---

## 문제 12 — 운영팀 공지만 필터링하기

department가 운영팀인 공지만 골라 날짜와 제목을 출력한다.

**기대 결과 형태**: 운영팀 공지만 나온다.

**빈칸 힌트**: 코드 셀의 빈칸을 채운다. selector 문자열과 딕셔너리 key는 lecture 노트북의 예제와 HTML 구조를 같이 보고 결정한다.

~~~python
# TODO: ____ 부분을 직접 채우세요.
ops_notices = [n for n in notices if n['____'] == '____']
for n in ops_notices:
    print(n['____'], n['____'])
~~~

---

## 문제 13 — 조회수 상위 공지 3개 찾기

views 기준으로 notices를 내림차순 정렬해 상위 3개 제목과 조회수를 출력한다.

**기대 결과 형태**: 조회수 높은 순서로 3개가 나온다.

**빈칸 힌트**: 코드 셀의 빈칸을 채운다. selector 문자열과 딕셔너리 key는 lecture 노트북의 예제와 HTML 구조를 같이 보고 결정한다.

~~~python
# TODO: ____ 부분을 직접 채우세요.
top3 = sorted(notices, key=lambda n: n['____'], reverse=____)[:____]
for n in top3:
    print(n['____'], n['____'])
~~~

---

## 문제 14 — targets.csv에서 허용 대상만 읽기

targets.csv를 csv.DictReader로 읽고 allowed가 yes인 행만 출력한다.

**기대 결과 형태**: 허용 대상 파일 목록이 출력된다.

**빈칸 힌트**: 코드 셀의 빈칸을 채운다. selector 문자열과 딕셔너리 key는 lecture 노트북의 예제와 HTML 구조를 같이 보고 결정한다.

~~~python
# TODO: ____ 부분을 직접 채우세요.
import io
reader = csv.DictReader(io.StringIO(target_csv))
allowed = []
for row in reader:
    if row['____'] == '____':
        allowed.append(row)
for row in allowed:
    print(row['____'], row['____'])
~~~

---

## 문제 15 — 수집 요약 문장 만들기

상품/공지 데이터를 근거로 한 문장 요약 3개를 만든다.

**기대 결과 형태**: 상품 수, 품절 수, 조회수 상위 공지에 대한 문장이 출력된다.

**빈칸 힌트**: 코드 셀의 빈칸을 채운다. selector 문자열과 딕셔너리 key는 lecture 노트북의 예제와 HTML 구조를 같이 보고 결정한다.

~~~python
# TODO: ____ 부분을 직접 채우세요.
summary1 = f"총 상품 수는 {____}개입니다."
summary2 = f"품절 상품은 {____}개입니다."
summary3 = f"조회수 1위 공지는 '{____}'입니다."
print(summary1)
print(summary2)
print(summary3)
~~~

---

## 제출 전 셀프 체크

아래 항목을 스스로 확인한 뒤 제출한다. 이 표는 정답을 알려주기 위한 것이 아니라, 노트북이 위에서부터 다시 실행되어도 같은 결과가 나오는지 확인하기 위한 점검표다.

| 체크 항목 | 확인 방법 |
|---|---|
| 환경 셀을 먼저 실행했다 | `data base:` 출력이 보이는지 확인한다 |
| HTML 원문을 읽었다 | `len(shop_html)`, `len(notice_html)` 이 0보다 큰지 확인한다 |
| 반복 단위 개수를 먼저 확인했다 | 상품은 `.product-card`, 공지는 `li.notice-item` 개수를 출력한다 |
| selector 결과가 비어 있지 않다 | `select_one(...)` 결과가 `None` 이 아닌지 확인한다 |
| 숫자 변환을 했다 | 가격, 재고, 조회수는 `int` 또는 `float` 로 바뀌었는지 확인한다 |
| 링크를 절대 URL로 만들었다 | `detail_url` 이 `https://` 로 시작하는지 확인한다 |
| CSV 저장이 끝났다 | `lesson01_products.csv` 파일명이 출력되는지 확인한다 |

## 디버깅 규칙

문제가 막히면 긴 코드를 한 번에 고치지 않는다. 아래 순서대로 한 줄씩 확인한다.

1. HTML을 제대로 읽었는가?
2. BeautifulSoup 객체를 만들었는가?
3. 반복 단위 selector 개수가 맞는가?
4. 첫 번째 반복 단위에서 내부 selector가 잡히는가?
5. 문자열에서 숫자를 꺼내는 함수가 필요한가?
6. 반복문 안에서 만든 딕셔너리 key 이름이 뒤 문제와 같은가?

예를 들어 문제 8~10이 모두 실패하면 가격 조건식만 볼 것이 아니라 문제 6의 `products` 구조부터 다시 확인한다. `products[0]` 을 출력했을 때 `price`, `rating`, `stock` key가 없거나 문자열로 남아 있으면 뒤 문제는 모두 흔들린다.

~~~python
# 제출 전 선택 점검 셀: 필요할 때만 실행하세요.
print('products type:', type(products))
print('products count:', len(products))
print('first product:', products[0] if products else '비어 있음')
print('notices count:', len(notices) if 'notices' in globals() else 'notices 없음')
~~~

## 좋은 제출물 기준

좋은 제출물은 정답 숫자만 맞는 노트북이 아니다. 다른 사람이 열어도 "어떤 HTML에서 무엇을 뽑았고, 어떤 기준으로 필터링했는지"를 따라갈 수 있어야 한다.

- 변수명은 `a`, `b`, `x` 보다 `products`, `notices`, `sold_out` 처럼 의미 있게 쓴다.
- 중간 결과는 너무 많이 출력하지 말고, 개수와 첫 번째 샘플 정도만 확인한다.
- 마지막 요약 문장은 출력된 숫자와 맞아야 한다.
- 실패했던 임시 코드는 제출 전에 정리한다.
- 실제 사이트에 적용하겠다는 문장을 쓸 때는 robots.txt, 약관, 요청 간격 확인을 함께 적는다.

## 다음 레슨 준비 질문

2강에서는 한 페이지가 아니라 여러 URL을 순서대로 읽는다. 오늘 만든 코드에서 다음 질문에 답할 수 있으면 2강 준비가 된 것이다.

1. 파일 이름만 바뀌어도 `load_text()` 로 같은 방식으로 읽을 수 있는가?
2. 반복 단위 selector가 페이지마다 같다면 같은 함수를 재사용할 수 있는가?
3. 상대 URL을 절대 URL로 바꾸지 않으면 여러 페이지 결과를 합칠 때 어떤 문제가 생기는가?
4. 요청 간격을 어디에 넣어야 여러 페이지 처리에서도 안전한가?
