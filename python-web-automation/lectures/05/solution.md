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

[![Open in Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/Kevin-innovation/jupyter-lecture/blob/main/python-web-automation/lectures/05/%EB%A0%88%EC%8A%A8%2005%20%E2%80%94%20%EC%84%B8%EC%85%98%EA%B3%BC%20%EC%BF%A0%ED%82%A4%20%EC%83%81%ED%83%9C%20%EA%B4%80%EB%A6%AC.ipynb)

세션과 쿠키 상태 관리 실습 문제의 모범 답안이다. 출력만 맞는지보다 상태 변화 순서, 쿠키 key, 장바구니 수량, 저장 산출물이 안전 기준과 맞는지 확인한다.

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

def read_soup(filename):
    return BeautifulSoup(load_text(filename), 'html.parser')

def clean_int(text):
    return int(re.sub(r'[^0-9]', '', str(text)))

def read_csv_rows(filename):
    return list(csv.DictReader(load_text(filename).splitlines()))

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

    def logout(self):
        self.client.cookies.clear()
        self.log.append({'event': 'logout'})

    def set_filter(self, name, value):
        self.client.cookies.set(f'filter_{name}', str(value))
        self.log.append({'event': 'filter', 'name': name, 'value': str(value)})

    def add_to_cart(self, product_id, quantity=1):
        quantity = int(quantity)
        self.cart.append({'product_id': product_id, 'quantity': quantity})
        self.log.append({'event': 'add', 'product_id': product_id, 'quantity': quantity})

    def remove_from_cart(self, product_id, quantity=1):
        quantity = int(quantity)
        remaining = quantity
        new_cart = []
        for item in self.cart:
            if item['product_id'] == product_id and remaining > 0:
                removed = min(item['quantity'], remaining)
                item = {**item, 'quantity': item['quantity'] - removed}
                remaining -= removed
            if item['quantity'] > 0:
                new_cart.append(item)
        self.cart = new_cart
        self.log.append({'event': 'remove', 'product_id': product_id, 'quantity': quantity - remaining})

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

로그인 폼 HTML을 정상적으로 읽었는지 확인하는 첫 단계다. BeautifulSoup 객체로 바꾼 뒤 h1을 읽으면 파일 경로와 파싱이 모두 정상인지 빠르게 확인할 수 있다.

### 채점 포인트

- 폼 구조와 hidden token을 정확히 읽는지 확인한다.
- 상태 변화가 출력값과 로그에 모두 반영되는지 확인한다.
- 이전 셀 변수를 재사용하는 문제는 실행 순서도 함께 확인한다.

### 자주 보이는 오답

- input text만 보고 hidden token을 놓치는 경우가 있다.
- 출력은 비슷하지만 실제 session.log가 남지 않는다.
- 실제 계정이나 실제 쿠키를 예제로 사용한다.

---

## 문제 2 정답 — hidden CSRF 토큰 읽기

~~~python
token = soup.select_one('input[name="csrf_token"]')['value']
print(token)
~~~

### 왜 이 코드가 정답인지

hidden input은 화면에 보이지 않지만 폼 제출 흐름에서 필요한 값이다. selector로 해당 input을 찾고 value 속성을 읽으면 token을 확인할 수 있다.

### 채점 포인트

- DemoSession 상태가 쿠키와 로그에 함께 반영되는지 확인한다.
- 상태 변화가 출력값과 로그에 모두 반영되는지 확인한다.
- 이전 셀 변수를 재사용하는 문제는 실행 순서도 함께 확인한다.

### 자주 보이는 오답

- session 객체를 새로 만들어 이전 상태를 잃는 경우가 있다.
- 출력은 비슷하지만 실제 session.log가 남지 않는다.
- 실제 계정이나 실제 쿠키를 예제로 사용한다.

---

## 문제 3 정답 — 폼 입력 name 목록 추출

~~~python
inputs = soup.select('input')
names = [tag.get('name') for tag in inputs]
print(names)
~~~

### 왜 이 코드가 정답인지

폼 자동화에서는 각 input의 name을 알아야 payload를 만들 수 있다. 모든 input을 선택하고 name 속성을 모으면 필요한 필드 목록을 확인할 수 있다.

### 채점 포인트

- 카탈로그 카드의 data 속성과 화면 텍스트를 구분하는지 확인한다.
- 상태 변화가 출력값과 로그에 모두 반영되는지 확인한다.
- 이전 셀 변수를 재사용하는 문제는 실행 순서도 함께 확인한다.

### 자주 보이는 오답

- data-price를 문자열로 두어 계산이 깨지는 경우가 있다.
- 출력은 비슷하지만 실제 session.log가 남지 않는다.
- 실제 계정이나 실제 쿠키를 예제로 사용한다.

---

## 문제 4 정답 — DemoSession 생성과 로그인

~~~python
session = DemoSession()
session.login('student01', token)
print(session.is_logged_in())
print(session.cookies())
~~~

### 왜 이 코드가 정답인지

DemoSession.login은 token이 맞을 때 합성 쿠키를 설정한다. is_logged_in과 cookies를 함께 출력하면 상태가 객체 안에 유지되는지 확인할 수 있다.

### 채점 포인트

- 장바구니 product_id와 catalog 가격표가 연결되는지 확인한다.
- 상태 변화가 출력값과 로그에 모두 반영되는지 확인한다.
- 이전 셀 변수를 재사용하는 문제는 실행 순서도 함께 확인한다.

### 자주 보이는 오답

- remove 이벤트를 처리하지 않아 cart_count가 맞지 않는 경우가 있다.
- 출력은 비슷하지만 실제 session.log가 남지 않는다.
- 실제 계정이나 실제 쿠키를 예제로 사용한다.

---

## 문제 5 정답 — 카탈로그 카드 개수 세기

~~~python
catalog = BeautifulSoup(load_text('catalog_page.html'), 'html.parser')
cards = catalog.select('article.product-card')
print(len(cards))
~~~

### 왜 이 코드가 정답인지

article.product-card는 카탈로그의 상품 하나를 나타내는 반복 단위다. 개수를 먼저 확인하면 selector가 정확한지 알 수 있다.

### 채점 포인트

- 쿠키 값 원문 대신 안전한 요약만 저장하는지 확인한다.
- 상태 변화가 출력값과 로그에 모두 반영되는지 확인한다.
- 이전 셀 변수를 재사용하는 문제는 실행 순서도 함께 확인한다.

### 자주 보이는 오답

- 쿠키 전체 값이나 token을 산출물에 저장하는 경우가 있다.
- 출력은 비슷하지만 실제 session.log가 남지 않는다.
- 실제 계정이나 실제 쿠키를 예제로 사용한다.

---

## 문제 6 정답 — 첫 상품 정보 읽기

~~~python
first = cards[0]
print(first['data-product-id'])
print(first.select_one('.name').text.strip())
print(first['data-price'])
~~~

### 왜 이 코드가 정답인지

상품 정보는 텍스트와 속성에 나뉘어 있다. product-id와 price는 계산과 상태 연결에 쓰는 값이므로 data 속성에서 읽는다.

### 채점 포인트

- 폼 구조와 hidden token을 정확히 읽는지 확인한다.
- 상태 변화가 출력값과 로그에 모두 반영되는지 확인한다.
- 이전 셀 변수를 재사용하는 문제는 실행 순서도 함께 확인한다.

### 자주 보이는 오답

- input text만 보고 hidden token을 놓치는 경우가 있다.
- 출력은 비슷하지만 실제 session.log가 남지 않는다.
- 실제 계정이나 실제 쿠키를 예제로 사용한다.

---

## 문제 7 정답 — 카테고리 필터 쿠키 저장

~~~python
session.set_filter('category', 'book')
print(session.cookies()['filter_category'])
~~~

### 왜 이 코드가 정답인지

필터 선택은 화면 상태를 유지하는 대표적인 예다. set_filter가 filter_category 쿠키를 만들고 이후 상품 필터링에서 같은 값을 사용할 수 있다.

### 채점 포인트

- DemoSession 상태가 쿠키와 로그에 함께 반영되는지 확인한다.
- 상태 변화가 출력값과 로그에 모두 반영되는지 확인한다.
- 이전 셀 변수를 재사용하는 문제는 실행 순서도 함께 확인한다.

### 자주 보이는 오답

- session 객체를 새로 만들어 이전 상태를 잃는 경우가 있다.
- 출력은 비슷하지만 실제 session.log가 남지 않는다.
- 실제 계정이나 실제 쿠키를 예제로 사용한다.

---

## 문제 8 정답 — 필터에 맞는 상품만 선택

~~~python
selected = [card for card in cards if card['data-category'] == session.cookies()['filter_category']]
print(len(selected))
print(selected[0].select_one('.name').text.strip())
~~~

### 왜 이 코드가 정답인지

쿠키에 저장된 필터 값과 카드의 category 속성을 비교하면 현재 화면 상태에 맞는 상품만 남길 수 있다.

### 채점 포인트

- 카탈로그 카드의 data 속성과 화면 텍스트를 구분하는지 확인한다.
- 상태 변화가 출력값과 로그에 모두 반영되는지 확인한다.
- 이전 셀 변수를 재사용하는 문제는 실행 순서도 함께 확인한다.

### 자주 보이는 오답

- data-price를 문자열로 두어 계산이 깨지는 경우가 있다.
- 출력은 비슷하지만 실제 session.log가 남지 않는다.
- 실제 계정이나 실제 쿠키를 예제로 사용한다.

---

## 문제 9 정답 — 장바구니에 상품 담기

~~~python
session.add_to_cart(selected[0]['data-product-id'], quantity=2)
print(session.cart)
print(session.cart_count())
~~~

### 왜 이 코드가 정답인지

장바구니 상태는 세션 객체 안의 리스트에 저장된다. product_id와 quantity를 남기면 이후 가격표와 연결해 총액을 계산할 수 있다.

### 채점 포인트

- 장바구니 product_id와 catalog 가격표가 연결되는지 확인한다.
- 상태 변화가 출력값과 로그에 모두 반영되는지 확인한다.
- 이전 셀 변수를 재사용하는 문제는 실행 순서도 함께 확인한다.

### 자주 보이는 오답

- remove 이벤트를 처리하지 않아 cart_count가 맞지 않는 경우가 있다.
- 출력은 비슷하지만 실제 session.log가 남지 않는다.
- 실제 계정이나 실제 쿠키를 예제로 사용한다.

---

## 문제 10 정답 — 장바구니 금액 계산

~~~python
price_by_id = {card['data-product-id']: int(card['data-price']) for card in cards}
name_by_id = {card['data-product-id']: card.select_one('.name').text.strip() for card in cards}
total = sum(price_by_id[item['product_id']] * item['quantity'] for item in session.cart)
print(total)
~~~

### 왜 이 코드가 정답인지

장바구니에는 상품 id와 수량만 있으므로 가격은 catalog에서 만든 price_by_id로 조회한다. data-price를 숫자로 바꾸어야 곱셈이 가능하다.

### 채점 포인트

- 쿠키 값 원문 대신 안전한 요약만 저장하는지 확인한다.
- 상태 변화가 출력값과 로그에 모두 반영되는지 확인한다.
- 이전 셀 변수를 재사용하는 문제는 실행 순서도 함께 확인한다.

### 자주 보이는 오답

- 쿠키 전체 값이나 token을 산출물에 저장하는 경우가 있다.
- 출력은 비슷하지만 실제 session.log가 남지 않는다.
- 실제 계정이나 실제 쿠키를 예제로 사용한다.

---

## 문제 11 정답 — 쿠키 정책 문장 읽기

~~~python
policy_lines = [line.strip() for line in load_text('cookie_policy.txt').splitlines() if line.strip()]
print(policy_lines)
~~~

### 왜 이 코드가 정답인지

쿠키 정책은 코드보다 먼저 확인해야 할 안전 기준이다. 빈 줄을 제외하고 문장 리스트로 만들면 제출 메모나 요약에 재사용할 수 있다.

### 채점 포인트

- 폼 구조와 hidden token을 정확히 읽는지 확인한다.
- 상태 변화가 출력값과 로그에 모두 반영되는지 확인한다.
- 이전 셀 변수를 재사용하는 문제는 실행 순서도 함께 확인한다.

### 자주 보이는 오답

- input text만 보고 hidden token을 놓치는 경우가 있다.
- 출력은 비슷하지만 실제 session.log가 남지 않는다.
- 실제 계정이나 실제 쿠키를 예제로 사용한다.

---

## 문제 12 정답 — 세션 이벤트 CSV 읽기

~~~python
events = read_csv_rows('session_events.csv')
print(events[:3])
print(Counter(row['event'] for row in events))
~~~

### 왜 이 코드가 정답인지

이벤트 CSV는 상태 변화 순서를 담고 있다. event 컬럼을 세면 login, add, remove 같은 동작이 몇 번 있었는지 확인할 수 있다.

### 채점 포인트

- DemoSession 상태가 쿠키와 로그에 함께 반영되는지 확인한다.
- 상태 변화가 출력값과 로그에 모두 반영되는지 확인한다.
- 이전 셀 변수를 재사용하는 문제는 실행 순서도 함께 확인한다.

### 자주 보이는 오답

- session 객체를 새로 만들어 이전 상태를 잃는 경우가 있다.
- 출력은 비슷하지만 실제 session.log가 남지 않는다.
- 실제 계정이나 실제 쿠키를 예제로 사용한다.

---

## 문제 13 정답 — 이벤트 로그 재생하기

~~~python
replay = DemoSession()
replay.login('replay-user', token)
for row in events:
    if row['event'] == 'add':
        replay.add_to_cart(row['product_id'], int(row['quantity']))
    elif row['event'] == 'remove':
        replay.remove_from_cart(row['product_id'], int(row['quantity']))
print(replay.cart)
print(replay.cart_count())
~~~

### 왜 이 코드가 정답인지

이벤트를 순서대로 적용하면 최종 장바구니 상태를 재현할 수 있다. add와 remove가 같은 product_id를 대상으로 작동하는지 확인하는 훈련이다.

### 채점 포인트

- 카탈로그 카드의 data 속성과 화면 텍스트를 구분하는지 확인한다.
- 상태 변화가 출력값과 로그에 모두 반영되는지 확인한다.
- 이전 셀 변수를 재사용하는 문제는 실행 순서도 함께 확인한다.

### 자주 보이는 오답

- data-price를 문자열로 두어 계산이 깨지는 경우가 있다.
- 출력은 비슷하지만 실제 session.log가 남지 않는다.
- 실제 계정이나 실제 쿠키를 예제로 사용한다.

---

## 문제 14 정답 — 세션 로그 CSV 저장하기

~~~python
output_dir = Path('outputs/lesson05')
output_dir.mkdir(parents=True, exist_ok=True)
log_path = output_dir / 'session_log.csv'
fieldnames = sorted({key for row in session.log for key in row.keys()})
with log_path.open('w', newline='', encoding='utf-8') as f:
    writer = csv.DictWriter(f, fieldnames=fieldnames)
    writer.writeheader()
    writer.writerows(session.log)
print(log_path, len(session.log))
~~~

### 왜 이 코드가 정답인지

세션 상태 변화는 CSV 로그로 남기면 나중에 순서를 다시 볼 수 있다. fieldnames를 로그 key 전체에서 만들면 event마다 key가 조금 달라도 저장할 수 있다.

### 채점 포인트

- 장바구니 product_id와 catalog 가격표가 연결되는지 확인한다.
- 상태 변화가 출력값과 로그에 모두 반영되는지 확인한다.
- 이전 셀 변수를 재사용하는 문제는 실행 순서도 함께 확인한다.

### 자주 보이는 오답

- remove 이벤트를 처리하지 않아 cart_count가 맞지 않는 경우가 있다.
- 출력은 비슷하지만 실제 session.log가 남지 않는다.
- 실제 계정이나 실제 쿠키를 예제로 사용한다.

---

## 문제 15 정답 — 상태 요약 JSON 저장하기

~~~python
summary = {'logged_in': session.is_logged_in(), 'cookie_keys': sorted(session.cookies().keys()), 'cart_count': session.cart_count(), 'policy_count': len(policy_lines)}
summary_path = output_dir / 'session_summary.json'
summary_path.write_text(json.dumps(summary, ensure_ascii=False, indent=2), encoding='utf-8')
print(summary)
print(summary_path)
~~~

### 왜 이 코드가 정답인지

요약 JSON은 현재 상태를 빠르게 보는 산출물이다. 민감한 쿠키 값을 저장하지 않고 key 목록만 남기면 상태 구조를 확인하면서도 안전 기준을 지킬 수 있다.

### 채점 포인트

- 쿠키 값 원문 대신 안전한 요약만 저장하는지 확인한다.
- 상태 변화가 출력값과 로그에 모두 반영되는지 확인한다.
- 이전 셀 변수를 재사용하는 문제는 실행 순서도 함께 확인한다.

### 자주 보이는 오답

- 쿠키 전체 값이나 token을 산출물에 저장하는 경우가 있다.
- 출력은 비슷하지만 실제 session.log가 남지 않는다.
- 실제 계정이나 실제 쿠키를 예제로 사용한다.

---

## 교사용 상세 피드백 기준

### 폼과 token 단계

문제 1~3은 로그인 자동화를 만들기 위한 수업이 아니라 폼 구조를 읽는 연습이다. 학생이 hidden input의 의미를 설명할 수 있어야 한다. 실제 서비스 token을 저장하거나 제출하지 않는다는 원칙도 함께 확인한다.

### 세션과 쿠키 단계

문제 4~8은 같은 DemoSession 객체 안에서 상태가 이어지는지 보는 구간이다. 학생이 중간에 session = DemoSession()을 다시 만들면 이전 쿠키와 로그가 사라진다. 상태가 이어져야 하는 코드와 초기화되어야 하는 코드를 구분하게 한다.

### 장바구니와 이벤트 단계

문제 9~13은 상태 변화 순서를 검증한다. 장바구니는 product_id와 quantity만 저장하고, 가격과 이름은 catalog에서 조회하는 구조가 좋다. 이벤트 CSV를 재생할 때 add와 remove를 순서대로 처리하지 않으면 최종 수량이 달라진다.

### 저장 단계

문제 14~15는 산출물 안전성이 핵심이다. session_log.csv에는 이벤트 기록을 남기되 token이나 session_id 원문을 저장하지 않는다. session_summary.json에는 쿠키 key 목록, cart_count, 정책 문장 수처럼 검수에 필요한 최소 정보만 남긴다.

## 문제별 빠른 확인표

| 문제 | 핵심 검수 | 통과 기준 |
|---:|---|---|
| 1 | 폼 HTML 로드 | 제목 출력 |
| 2 | hidden token | safe-token-05 출력 |
| 3 | input name | 필드명 목록 출력 |
| 4 | 로그인 상태 | 쿠키와 True 출력 |
| 5 | 상품 카드 | 카드 10개 확인 |
| 6 | 상품 속성 | id, name, price 출력 |
| 7 | 필터 쿠키 | filter_category 생성 |
| 8 | 필터 적용 | book 상품만 선택 |
| 9 | 장바구니 | cart_count 증가 |
| 10 | 총액 계산 | 숫자 총액 출력 |
| 11 | 정책 읽기 | 안전 문장 리스트 출력 |
| 12 | 이벤트 CSV | event별 개수 출력 |
| 13 | 이벤트 재생 | 최종 cart 설명 가능 |
| 14 | 로그 CSV | session_log.csv 생성 |
| 15 | 요약 JSON | cookie_keys만 저장 |

## 엄격 채점이 필요한 항목

- 실제 서비스 쿠키나 계정 정보를 사용한 답안은 통과시키지 않는다.
- 비밀번호, token, session_id 원문을 산출물에 저장하면 보완을 요구한다.
- 쿠키 key와 쿠키 value를 구분하지 못하면 문제 15는 통과로 보지 않는다.
- 이벤트 로그를 순서대로 처리하지 않은 답안은 최종 상태 검증을 다시 요구한다.
- 저장 파일이 없거나 새 런타임에서 실행되지 않으면 최종 미션 완료로 보지 않는다.

## 문제별 오답 대응 가이드

### 문제 1~3

폼 구조를 읽는 단계에서는 token 값을 찾는 selector가 핵심이다. 학생이 username, password만 보고 hidden input을 놓치면 실제 폼 흐름을 이해하지 못한 것이다. 다만 token 원문을 산출물에 저장하면 안 된다는 점을 함께 지도한다.

### 문제 4~8

DemoSession을 한 번 만들고 그 객체를 계속 써야 상태가 이어진다. 학생이 문제마다 새 session을 만들면 앞에서 설정한 쿠키가 사라진다. 쿠키 key가 없어서 KeyError가 나는 경우는 대부분 set_filter를 실행하지 않았거나 다른 객체를 사용했기 때문이다.

### 문제 9~10

장바구니 총액 계산에서는 product_id를 연결 키로 사용한다. 상품명으로 연결하면 이름이 바뀌었을 때 깨질 수 있고, 가격 문자열로 계산하면 쉼표나 원 문자 때문에 오류가 난다. data-price를 int로 바꿔 price_by_id를 만드는 방식이 안정적이다.

### 문제 11~13

정책 파일은 단순 텍스트지만 수업 안전 기준을 확인하는 자료다. 이벤트 CSV는 상태 변화 순서가 중요하다. remove 이벤트를 처리하지 않으면 최종 장바구니 수량이 틀어지고, logout 이벤트를 처리하지 않으면 쿠키 상태가 실제 로그와 다르게 남는다.

### 문제 14~15

CSV 로그는 event별 key가 다를 수 있다. fieldnames를 고정으로 쓰면 일부 key가 빠질 수 있으므로 로그 rows 전체 key를 모아 만드는 방식이 좋다. JSON 요약에는 쿠키 value가 아니라 cookie_keys를 저장해야 한다. 이 차이를 설명하지 못하면 세션 안전 기준을 통과한 것으로 보기 어렵다.

## 산출물 안전 검수

검수자는 session_log.csv와 session_summary.json을 열어 민감정보가 들어가지 않았는지 확인한다. session_summary.json에 demo-session-05 같은 값이 직접 들어가 있으면 cookie value를 저장한 것이므로 수정해야 한다. cookie_keys에 session_id라는 문자열이 들어가는 것은 구조 요약이므로 허용할 수 있다.

## 우수 답안 기준

우수 답안은 상태 변화 함수를 잘게 나누고, replay 결과와 현재 session 결과를 분리해 비교한다. 이벤트 로그에서 checkout과 logout을 별도 상태로 기록하거나, cart_total을 replay에도 계산하면 더 좋다. 단, 기능이 늘어나도 민감정보 저장 금지와 fixture 사용 원칙은 지켜야 한다.

## 재실행 검수 기준

교사용 검수에서는 outputs/lesson05 폴더를 삭제한 뒤 새 런타임에서 전체 실행한다. 같은 CSV와 JSON이 만들어지고, JSON에 cookie_keys만 남으면 기본 안정성을 통과한 것이다. 중간 셀 실행 결과에 의존하는 답안은 실제 운영 자동화로 보기 어렵다.

