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

# 레슨 05 — 실습 문제 정답지

> 🔒 교사·관리자 전용. 학생에게 배포 금지.

세션과 쿠키 상태 관리 실습 문제의 모범 답안이다. 출력값만 보지 말고 상태 변화, selector 안정성, 로그를 함께 확인한다.

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
    DATA_BASE = 'https://raw.githubusercontent.com/Kevin-innovation/jupyter-lecture/main/python-web-automation/lectures/05/data'
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

def clean_int(text):
    return int(re.sub(r'[^0-9]', '', str(text)))

class DemoSession:
    def __init__(self):
        self.client = requests.Session()
        self.cart = []
        self.log = []
    def login(self, username, token):
        if token != 'safe-token-05':
            raise ValueError('invalid token')
        self.client.cookies.set('demo_user', username)
        self.client.cookies.set('session_id', 'demo-session-05')
        self.log.append({'event': 'login', 'user': username})
    def set_filter(self, name, value):
        self.client.cookies.set(f'filter_{name}', str(value))
        self.log.append({'event': 'filter', 'name': name, 'value': str(value)})
    def add_to_cart(self, product_id, quantity=1):
        self.cart.append({'product_id': product_id, 'quantity': int(quantity)})
        self.log.append({'event': 'add', 'product_id': product_id, 'quantity': int(quantity)})
    def is_logged_in(self):
        return self.client.cookies.get('session_id') is not None
    def cookies(self):
        return self.client.cookies.get_dict()
    def cart_count(self):
        return sum(item['quantity'] for item in self.cart)

print('colab:', IS_COLAB)
print('data base:', DATA_BASE)
~~~

---

## 문제 1 정답 — 로그인 폼 제목 읽기

~~~python
form_html = load_text('login_form.html')
soup = BeautifulSoup(form_html, 'html.parser')
print(soup.select_one('h1').text.strip())
~~~

### 왜 이 코드가 정답인지

이 답안은 문제에서 요구한 상태나 화면 구조를 먼저 명확한 자료구조로 바꾼 뒤 필요한 값만 선택한다. 로그인 HTML을 soup 객체로 바꿔야 hidden token과 input 구조를 읽을 수 있다. 결과가 맞아도 세션 상태, locator, wait, 로그가 코드에 남아 있지 않으면 운영형 자동화로 보기 어렵다.

### 채점 포인트

- 입력 fixture를 환경 셀의 헬퍼로 읽어 코랩과 로컬에서 모두 동작하는가.
- 상태 변화, 버튼 클릭, wait 조건을 중간 변수나 로그로 확인할 수 있는가.
- selector가 화면 문구보다 id, data-testid, role 같은 안정적인 기준을 우선하는가.
- 출력 형태가 문제 요구사항과 일치하고 CSV 저장 시 헤더가 있는가.

### 자주 보이는 오답

- 화면 텍스트만 믿고 불안정한 selector를 사용한다.
- 상태가 바뀌기 전에 결과를 읽어 빈 값이나 이전 값이 나온다.
- 쿠키나 폼 입력 값을 문자열 변수에만 두고 세션/페이지 상태에 반영하지 않는다.
- 최종 미션에서 개별 문제 코드를 복사만 해서 함수화와 로그가 빠진다.

---

## 문제 2 정답 — hidden CSRF 토큰 읽기

~~~python
token = soup.select_one('input[name="csrf_token"]')['value']
print(token)
~~~

### 왜 이 코드가 정답인지

이 답안은 문제에서 요구한 상태나 화면 구조를 먼저 명확한 자료구조로 바꾼 뒤 필요한 값만 선택한다. 토큰은 화면에 보이지 않지만 form 전송 상태를 확인하는 핵심 값이다. 결과가 맞아도 세션 상태, locator, wait, 로그가 코드에 남아 있지 않으면 운영형 자동화로 보기 어렵다.

### 채점 포인트

- 입력 fixture를 환경 셀의 헬퍼로 읽어 코랩과 로컬에서 모두 동작하는가.
- 상태 변화, 버튼 클릭, wait 조건을 중간 변수나 로그로 확인할 수 있는가.
- selector가 화면 문구보다 id, data-testid, role 같은 안정적인 기준을 우선하는가.
- 출력 형태가 문제 요구사항과 일치하고 CSV 저장 시 헤더가 있는가.

### 자주 보이는 오답

- 화면 텍스트만 믿고 불안정한 selector를 사용한다.
- 상태가 바뀌기 전에 결과를 읽어 빈 값이나 이전 값이 나온다.
- 쿠키나 폼 입력 값을 문자열 변수에만 두고 세션/페이지 상태에 반영하지 않는다.
- 최종 미션에서 개별 문제 코드를 복사만 해서 함수화와 로그가 빠진다.

---

## 문제 3 정답 — 폼 입력 name 목록 추출

~~~python
inputs = soup.select('form input')
names = [tag.get('name') for tag in inputs]
print(names)
~~~

### 왜 이 코드가 정답인지

이 답안은 문제에서 요구한 상태나 화면 구조를 먼저 명확한 자료구조로 바꾼 뒤 필요한 값만 선택한다. 폼 자동화는 어떤 input name이 필요한지 먼저 확인해야 한다. 결과가 맞아도 세션 상태, locator, wait, 로그가 코드에 남아 있지 않으면 운영형 자동화로 보기 어렵다.

### 채점 포인트

- 입력 fixture를 환경 셀의 헬퍼로 읽어 코랩과 로컬에서 모두 동작하는가.
- 상태 변화, 버튼 클릭, wait 조건을 중간 변수나 로그로 확인할 수 있는가.
- selector가 화면 문구보다 id, data-testid, role 같은 안정적인 기준을 우선하는가.
- 출력 형태가 문제 요구사항과 일치하고 CSV 저장 시 헤더가 있는가.

### 자주 보이는 오답

- 화면 텍스트만 믿고 불안정한 selector를 사용한다.
- 상태가 바뀌기 전에 결과를 읽어 빈 값이나 이전 값이 나온다.
- 쿠키나 폼 입력 값을 문자열 변수에만 두고 세션/페이지 상태에 반영하지 않는다.
- 최종 미션에서 개별 문제 코드를 복사만 해서 함수화와 로그가 빠진다.

---

## 문제 4 정답 — DemoSession 생성과 로그인

~~~python
session = DemoSession()
session.login('student01', token)
print(session.is_logged_in())
print(session.cookies())
~~~

### 왜 이 코드가 정답인지

이 답안은 문제에서 요구한 상태나 화면 구조를 먼저 명확한 자료구조로 바꾼 뒤 필요한 값만 선택한다. 로그인 후 쿠키 jar에 demo_user와 session_id가 들어가야 세션 상태가 유지된다. 결과가 맞아도 세션 상태, locator, wait, 로그가 코드에 남아 있지 않으면 운영형 자동화로 보기 어렵다.

### 채점 포인트

- 입력 fixture를 환경 셀의 헬퍼로 읽어 코랩과 로컬에서 모두 동작하는가.
- 상태 변화, 버튼 클릭, wait 조건을 중간 변수나 로그로 확인할 수 있는가.
- selector가 화면 문구보다 id, data-testid, role 같은 안정적인 기준을 우선하는가.
- 출력 형태가 문제 요구사항과 일치하고 CSV 저장 시 헤더가 있는가.

### 자주 보이는 오답

- 화면 텍스트만 믿고 불안정한 selector를 사용한다.
- 상태가 바뀌기 전에 결과를 읽어 빈 값이나 이전 값이 나온다.
- 쿠키나 폼 입력 값을 문자열 변수에만 두고 세션/페이지 상태에 반영하지 않는다.
- 최종 미션에서 개별 문제 코드를 복사만 해서 함수화와 로그가 빠진다.

---

## 문제 5 정답 — 카탈로그 카드 개수 세기

~~~python
catalog = BeautifulSoup(load_text('catalog_page.html'), 'html.parser')
cards = catalog.select('article.product-card')
print(len(cards))
~~~

### 왜 이 코드가 정답인지

이 답안은 문제에서 요구한 상태나 화면 구조를 먼저 명확한 자료구조로 바꾼 뒤 필요한 값만 선택한다. 상품 카드가 장바구니 반복 단위이므로 먼저 개수를 확인한다. 결과가 맞아도 세션 상태, locator, wait, 로그가 코드에 남아 있지 않으면 운영형 자동화로 보기 어렵다.

### 채점 포인트

- 입력 fixture를 환경 셀의 헬퍼로 읽어 코랩과 로컬에서 모두 동작하는가.
- 상태 변화, 버튼 클릭, wait 조건을 중간 변수나 로그로 확인할 수 있는가.
- selector가 화면 문구보다 id, data-testid, role 같은 안정적인 기준을 우선하는가.
- 출력 형태가 문제 요구사항과 일치하고 CSV 저장 시 헤더가 있는가.

### 자주 보이는 오답

- 화면 텍스트만 믿고 불안정한 selector를 사용한다.
- 상태가 바뀌기 전에 결과를 읽어 빈 값이나 이전 값이 나온다.
- 쿠키나 폼 입력 값을 문자열 변수에만 두고 세션/페이지 상태에 반영하지 않는다.
- 최종 미션에서 개별 문제 코드를 복사만 해서 함수화와 로그가 빠진다.

---

## 문제 6 정답 — 첫 상품 정보 읽기

~~~python
first = cards[0]
print(first['data-product-id'])
print(first.select_one('.name').text.strip())
print(first['data-price'])
~~~

### 왜 이 코드가 정답인지

이 답안은 문제에서 요구한 상태나 화면 구조를 먼저 명확한 자료구조로 바꾼 뒤 필요한 값만 선택한다. 상품 id, 이름, 가격은 화면 텍스트와 속성에서 각각 읽는다. 결과가 맞아도 세션 상태, locator, wait, 로그가 코드에 남아 있지 않으면 운영형 자동화로 보기 어렵다.

### 채점 포인트

- 입력 fixture를 환경 셀의 헬퍼로 읽어 코랩과 로컬에서 모두 동작하는가.
- 상태 변화, 버튼 클릭, wait 조건을 중간 변수나 로그로 확인할 수 있는가.
- selector가 화면 문구보다 id, data-testid, role 같은 안정적인 기준을 우선하는가.
- 출력 형태가 문제 요구사항과 일치하고 CSV 저장 시 헤더가 있는가.

### 자주 보이는 오답

- 화면 텍스트만 믿고 불안정한 selector를 사용한다.
- 상태가 바뀌기 전에 결과를 읽어 빈 값이나 이전 값이 나온다.
- 쿠키나 폼 입력 값을 문자열 변수에만 두고 세션/페이지 상태에 반영하지 않는다.
- 최종 미션에서 개별 문제 코드를 복사만 해서 함수화와 로그가 빠진다.

---

## 문제 7 정답 — 카테고리 필터 쿠키 저장

~~~python
session.set_filter('category', 'book')
print(session.cookies()['filter_category'])
~~~

### 왜 이 코드가 정답인지

이 답안은 문제에서 요구한 상태나 화면 구조를 먼저 명확한 자료구조로 바꾼 뒤 필요한 값만 선택한다. 필터 상태가 쿠키에 남으면 다음 요청에서도 같은 조건을 유지한다고 설명할 수 있다. 결과가 맞아도 세션 상태, locator, wait, 로그가 코드에 남아 있지 않으면 운영형 자동화로 보기 어렵다.

### 채점 포인트

- 입력 fixture를 환경 셀의 헬퍼로 읽어 코랩과 로컬에서 모두 동작하는가.
- 상태 변화, 버튼 클릭, wait 조건을 중간 변수나 로그로 확인할 수 있는가.
- selector가 화면 문구보다 id, data-testid, role 같은 안정적인 기준을 우선하는가.
- 출력 형태가 문제 요구사항과 일치하고 CSV 저장 시 헤더가 있는가.

### 자주 보이는 오답

- 화면 텍스트만 믿고 불안정한 selector를 사용한다.
- 상태가 바뀌기 전에 결과를 읽어 빈 값이나 이전 값이 나온다.
- 쿠키나 폼 입력 값을 문자열 변수에만 두고 세션/페이지 상태에 반영하지 않는다.
- 최종 미션에서 개별 문제 코드를 복사만 해서 함수화와 로그가 빠진다.

---

## 문제 8 정답 — 필터에 맞는 상품만 선택

~~~python
selected = [card for card in cards if card['data-category'] == session.cookies()['filter_category']]
print(len(selected))
print(selected[0].select_one('.name').text.strip())
~~~

### 왜 이 코드가 정답인지

이 답안은 문제에서 요구한 상태나 화면 구조를 먼저 명확한 자료구조로 바꾼 뒤 필요한 값만 선택한다. 쿠키의 필터 값과 카드 속성을 비교해야 상태 기반 필터가 된다. 결과가 맞아도 세션 상태, locator, wait, 로그가 코드에 남아 있지 않으면 운영형 자동화로 보기 어렵다.

### 채점 포인트

- 입력 fixture를 환경 셀의 헬퍼로 읽어 코랩과 로컬에서 모두 동작하는가.
- 상태 변화, 버튼 클릭, wait 조건을 중간 변수나 로그로 확인할 수 있는가.
- selector가 화면 문구보다 id, data-testid, role 같은 안정적인 기준을 우선하는가.
- 출력 형태가 문제 요구사항과 일치하고 CSV 저장 시 헤더가 있는가.

### 자주 보이는 오답

- 화면 텍스트만 믿고 불안정한 selector를 사용한다.
- 상태가 바뀌기 전에 결과를 읽어 빈 값이나 이전 값이 나온다.
- 쿠키나 폼 입력 값을 문자열 변수에만 두고 세션/페이지 상태에 반영하지 않는다.
- 최종 미션에서 개별 문제 코드를 복사만 해서 함수화와 로그가 빠진다.

---

## 문제 9 정답 — 장바구니에 상품 담기

~~~python
session.add_to_cart(selected[0]['data-product-id'], quantity=2)
print(session.cart_count())
print(session.cart)
~~~

### 왜 이 코드가 정답인지

이 답안은 문제에서 요구한 상태나 화면 구조를 먼저 명확한 자료구조로 바꾼 뒤 필요한 값만 선택한다. 장바구니는 세션 상태의 대표 예시이며 quantity 합계를 확인해야 한다. 결과가 맞아도 세션 상태, locator, wait, 로그가 코드에 남아 있지 않으면 운영형 자동화로 보기 어렵다.

### 채점 포인트

- 입력 fixture를 환경 셀의 헬퍼로 읽어 코랩과 로컬에서 모두 동작하는가.
- 상태 변화, 버튼 클릭, wait 조건을 중간 변수나 로그로 확인할 수 있는가.
- selector가 화면 문구보다 id, data-testid, role 같은 안정적인 기준을 우선하는가.
- 출력 형태가 문제 요구사항과 일치하고 CSV 저장 시 헤더가 있는가.

### 자주 보이는 오답

- 화면 텍스트만 믿고 불안정한 selector를 사용한다.
- 상태가 바뀌기 전에 결과를 읽어 빈 값이나 이전 값이 나온다.
- 쿠키나 폼 입력 값을 문자열 변수에만 두고 세션/페이지 상태에 반영하지 않는다.
- 최종 미션에서 개별 문제 코드를 복사만 해서 함수화와 로그가 빠진다.

---

## 문제 10 정답 — 상품 id로 가격 찾기

~~~python
price_map = {card['data-product-id']: clean_int(card['data-price']) for card in cards}
print(price_map[session.cart[0]['product_id']])
~~~

### 왜 이 코드가 정답인지

이 답안은 문제에서 요구한 상태나 화면 구조를 먼저 명확한 자료구조로 바꾼 뒤 필요한 값만 선택한다. 장바구니 계산을 위해 product_id와 가격을 매핑한다. 결과가 맞아도 세션 상태, locator, wait, 로그가 코드에 남아 있지 않으면 운영형 자동화로 보기 어렵다.

### 채점 포인트

- 입력 fixture를 환경 셀의 헬퍼로 읽어 코랩과 로컬에서 모두 동작하는가.
- 상태 변화, 버튼 클릭, wait 조건을 중간 변수나 로그로 확인할 수 있는가.
- selector가 화면 문구보다 id, data-testid, role 같은 안정적인 기준을 우선하는가.
- 출력 형태가 문제 요구사항과 일치하고 CSV 저장 시 헤더가 있는가.

### 자주 보이는 오답

- 화면 텍스트만 믿고 불안정한 selector를 사용한다.
- 상태가 바뀌기 전에 결과를 읽어 빈 값이나 이전 값이 나온다.
- 쿠키나 폼 입력 값을 문자열 변수에만 두고 세션/페이지 상태에 반영하지 않는다.
- 최종 미션에서 개별 문제 코드를 복사만 해서 함수화와 로그가 빠진다.

---

## 문제 11 정답 — 장바구니 총액 계산

~~~python
total = 0
for item in session.cart:
    total += price_map[item['product_id']] * item['quantity']
print(total)
~~~

### 왜 이 코드가 정답인지

이 답안은 문제에서 요구한 상태나 화면 구조를 먼저 명확한 자료구조로 바꾼 뒤 필요한 값만 선택한다. 총액은 세션 cart 상태와 가격 map을 함께 사용해야 계산된다. 결과가 맞아도 세션 상태, locator, wait, 로그가 코드에 남아 있지 않으면 운영형 자동화로 보기 어렵다.

### 채점 포인트

- 입력 fixture를 환경 셀의 헬퍼로 읽어 코랩과 로컬에서 모두 동작하는가.
- 상태 변화, 버튼 클릭, wait 조건을 중간 변수나 로그로 확인할 수 있는가.
- selector가 화면 문구보다 id, data-testid, role 같은 안정적인 기준을 우선하는가.
- 출력 형태가 문제 요구사항과 일치하고 CSV 저장 시 헤더가 있는가.

### 자주 보이는 오답

- 화면 텍스트만 믿고 불안정한 selector를 사용한다.
- 상태가 바뀌기 전에 결과를 읽어 빈 값이나 이전 값이 나온다.
- 쿠키나 폼 입력 값을 문자열 변수에만 두고 세션/페이지 상태에 반영하지 않는다.
- 최종 미션에서 개별 문제 코드를 복사만 해서 함수화와 로그가 빠진다.

---

## 문제 12 정답 — 세션 이벤트 CSV 읽기

~~~python
events = list(csv.DictReader(load_text('session_events.csv').splitlines()))
print(len(events))
print(events[0]['event'])
~~~

### 왜 이 코드가 정답인지

이 답안은 문제에서 요구한 상태나 화면 구조를 먼저 명확한 자료구조로 바꾼 뒤 필요한 값만 선택한다. CSV 로그는 실제 세션 변화 검증에 쓰는 운영 기록이다. 결과가 맞아도 세션 상태, locator, wait, 로그가 코드에 남아 있지 않으면 운영형 자동화로 보기 어렵다.

### 채점 포인트

- 입력 fixture를 환경 셀의 헬퍼로 읽어 코랩과 로컬에서 모두 동작하는가.
- 상태 변화, 버튼 클릭, wait 조건을 중간 변수나 로그로 확인할 수 있는가.
- selector가 화면 문구보다 id, data-testid, role 같은 안정적인 기준을 우선하는가.
- 출력 형태가 문제 요구사항과 일치하고 CSV 저장 시 헤더가 있는가.

### 자주 보이는 오답

- 화면 텍스트만 믿고 불안정한 selector를 사용한다.
- 상태가 바뀌기 전에 결과를 읽어 빈 값이나 이전 값이 나온다.
- 쿠키나 폼 입력 값을 문자열 변수에만 두고 세션/페이지 상태에 반영하지 않는다.
- 최종 미션에서 개별 문제 코드를 복사만 해서 함수화와 로그가 빠진다.

---

## 문제 13 정답 — add 이벤트만 필터링

~~~python
adds = [row for row in events if row['event'] == 'add']
print(len(adds))
print([row['product_id'] for row in adds])
~~~

### 왜 이 코드가 정답인지

이 답안은 문제에서 요구한 상태나 화면 구조를 먼저 명확한 자료구조로 바꾼 뒤 필요한 값만 선택한다. event 컬럼을 기준으로 장바구니 담기만 골라야 한다. 결과가 맞아도 세션 상태, locator, wait, 로그가 코드에 남아 있지 않으면 운영형 자동화로 보기 어렵다.

### 채점 포인트

- 입력 fixture를 환경 셀의 헬퍼로 읽어 코랩과 로컬에서 모두 동작하는가.
- 상태 변화, 버튼 클릭, wait 조건을 중간 변수나 로그로 확인할 수 있는가.
- selector가 화면 문구보다 id, data-testid, role 같은 안정적인 기준을 우선하는가.
- 출력 형태가 문제 요구사항과 일치하고 CSV 저장 시 헤더가 있는가.

### 자주 보이는 오답

- 화면 텍스트만 믿고 불안정한 selector를 사용한다.
- 상태가 바뀌기 전에 결과를 읽어 빈 값이나 이전 값이 나온다.
- 쿠키나 폼 입력 값을 문자열 변수에만 두고 세션/페이지 상태에 반영하지 않는다.
- 최종 미션에서 개별 문제 코드를 복사만 해서 함수화와 로그가 빠진다.

---

## 문제 14 정답 — 쿠키 정책 줄 수 확인

~~~python
policy = load_text('cookie_policy.txt')
lines = [line for line in policy.splitlines() if line.strip()]
print(len(lines))
print(lines[0])
~~~

### 왜 이 코드가 정답인지

이 답안은 문제에서 요구한 상태나 화면 구조를 먼저 명확한 자료구조로 바꾼 뒤 필요한 값만 선택한다. 쿠키는 기술뿐 아니라 보관 원칙과 안전 규칙을 함께 확인해야 한다. 결과가 맞아도 세션 상태, locator, wait, 로그가 코드에 남아 있지 않으면 운영형 자동화로 보기 어렵다.

### 채점 포인트

- 입력 fixture를 환경 셀의 헬퍼로 읽어 코랩과 로컬에서 모두 동작하는가.
- 상태 변화, 버튼 클릭, wait 조건을 중간 변수나 로그로 확인할 수 있는가.
- selector가 화면 문구보다 id, data-testid, role 같은 안정적인 기준을 우선하는가.
- 출력 형태가 문제 요구사항과 일치하고 CSV 저장 시 헤더가 있는가.

### 자주 보이는 오답

- 화면 텍스트만 믿고 불안정한 selector를 사용한다.
- 상태가 바뀌기 전에 결과를 읽어 빈 값이나 이전 값이 나온다.
- 쿠키나 폼 입력 값을 문자열 변수에만 두고 세션/페이지 상태에 반영하지 않는다.
- 최종 미션에서 개별 문제 코드를 복사만 해서 함수화와 로그가 빠진다.

---

## 문제 15 정답 — 세션 로그 CSV 저장

~~~python
with open('lesson05_session_log.csv', 'w', newline='', encoding='utf-8') as f:
    writer = csv.DictWriter(f, fieldnames=['event', 'user', 'name', 'value', 'product_id', 'quantity'])
    writer.writeheader()
    writer.writerows(session.log)
print('saved:', len(session.log))
~~~

### 왜 이 코드가 정답인지

이 답안은 문제에서 요구한 상태나 화면 구조를 먼저 명확한 자료구조로 바꾼 뒤 필요한 값만 선택한다. 세션 변화는 결과만 출력하지 말고 로그로 남겨야 운영형 자동화가 된다. 결과가 맞아도 세션 상태, locator, wait, 로그가 코드에 남아 있지 않으면 운영형 자동화로 보기 어렵다.

### 채점 포인트

- 입력 fixture를 환경 셀의 헬퍼로 읽어 코랩과 로컬에서 모두 동작하는가.
- 상태 변화, 버튼 클릭, wait 조건을 중간 변수나 로그로 확인할 수 있는가.
- selector가 화면 문구보다 id, data-testid, role 같은 안정적인 기준을 우선하는가.
- 출력 형태가 문제 요구사항과 일치하고 CSV 저장 시 헤더가 있는가.

### 자주 보이는 오답

- 화면 텍스트만 믿고 불안정한 selector를 사용한다.
- 상태가 바뀌기 전에 결과를 읽어 빈 값이나 이전 값이 나온다.
- 쿠키나 폼 입력 값을 문자열 변수에만 두고 세션/페이지 상태에 반영하지 않는다.
- 최종 미션에서 개별 문제 코드를 복사만 해서 함수화와 로그가 빠진다.

---

### 보강 설명 1

레슨 05 정답 해설은 실행 결과만 맞추는 것이 아니라 재현 가능한 절차를 남기는 것이 핵심이다. 입력 파일, 반복 단위, selector 또는 상태 변수, 저장 경로를 분리하면 오류 위치를 빠르게 찾을 수 있다.

### 보강 설명 2

학생이 막히면 완성 코드를 보여주기보다 HTML 구조, CSV 헤더, 상태 변화 순서를 먼저 말로 설명하게 한다. 구조를 설명할 수 있으면 코드도 안정된다.

### 보강 설명 3

실제 자동화는 다음 주에도 다시 실행되어야 한다. 그래서 쿠키, 폼 입력, selector, wait, 로그 같은 운영 요소를 초반부터 명시적으로 다룬다.
