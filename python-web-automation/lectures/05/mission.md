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

# 레슨 05 — 실습 문제

[![Open in Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/Kevin-innovation/jupyter-lecture/blob/main/python-web-automation/lectures/05/%5B%ED%95%99%EC%83%9D%EC%9A%A9%5D%20%EB%A0%88%EC%8A%A8%2005%20%E2%80%94%20%EC%84%B8%EC%85%98%EA%B3%BC%20%EC%BF%A0%ED%82%A4%20%EC%83%81%ED%83%9C%20%EA%B4%80%EB%A6%AC.ipynb)

세션과 쿠키 상태 관리 레슨의 학생용 문제 노트북이다. 강의 노트북을 먼저 실행한 뒤 빈칸을 직접 채운다. 문제는 로그인 폼 구조 확인에서 시작해 쿠키 필터, 장바구니 상태, 이벤트 로그, 안전한 요약 저장까지 이어진다.

## 통과 기준

- 총 15문제 중 12문제 이상 정상 출력이면 통과.
- 문제 1~4는 로그인 폼과 세션 생성, 5~10은 카탈로그와 장바구니 상태, 11~15는 정책과 이벤트 로그 저장이다.
- 정답값과 완성 코드는 적지 않는다. 출력 형태, 상태 변화, 저장 파일 이름을 보고 직접 판단한다.

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

## 문제 1 — 로그인 폼 제목 읽기

login_form.html을 읽고 h1 제목을 출력한다.

**기대 결과 형태**: 로그인 폼 제목이 한 줄로 출력된다.

**빈칸 힌트**: 파일은 load_text로 읽고 BeautifulSoup으로 파싱한다.

~~~python
form_html = ____(____)
soup = ____(form_html, 'html.parser')
print(soup.____('____').text.strip())
~~~

---

## 문제 2 — hidden CSRF 토큰 읽기

로그인 폼의 hidden input에서 csrf_token 값을 읽는다.

**기대 결과 형태**: safe-token-05 값이 출력된다.

**빈칸 힌트**: input[name="csrf_token"]의 value 속성을 읽는다.

~~~python
token = soup.select_one('input[name="____"]')['____']
print(token)
~~~

---

## 문제 3 — 폼 입력 name 목록 추출

로그인 폼 안의 input name 값을 모두 리스트로 만든다.

**기대 결과 형태**: csrf_token, username, password가 포함된 리스트가 출력된다.

**빈칸 힌트**: input 태그를 모두 선택하고 get("name")을 사용한다.

~~~python
inputs = soup.select('____')
names = [tag.get('____') for tag in inputs]
print(names)
~~~

---

## 문제 4 — DemoSession 생성과 로그인

DemoSession 객체를 만들고 token으로 로그인 상태를 만든다.

**기대 결과 형태**: 로그인 여부와 쿠키 딕셔너리가 출력된다.

**빈칸 힌트**: login 메서드는 username과 token을 받는다.

~~~python
session = ____()
session.____('student01', token)
print(session.____())
print(session.____())
~~~

---

## 문제 5 — 카탈로그 카드 개수 세기

catalog_page.html에서 상품 카드 반복 단위를 선택한다.

**기대 결과 형태**: 상품 카드 개수가 출력된다.

**빈칸 힌트**: 상품 반복 단위는 article.product-card다.

~~~python
catalog = BeautifulSoup(load_text('____'), 'html.parser')
cards = catalog.select('____')
print(len(cards))
~~~

---

## 문제 6 — 첫 상품 정보 읽기

첫 상품의 id, 이름, 가격 속성을 읽는다.

**기대 결과 형태**: 상품 id, 상품명, data-price가 출력된다.

**빈칸 힌트**: id와 가격은 data 속성, 이름은 .name 텍스트다.

~~~python
first = cards[____]
print(first['____'])
print(first.select_one('____').text.strip())
print(first['____'])
~~~

---

## 문제 7 — 카테고리 필터 쿠키 저장

세션에 category 필터를 book으로 저장한다.

**기대 결과 형태**: filter_category 쿠키 값이 출력된다.

**빈칸 힌트**: set_filter는 name과 value를 받는다.

~~~python
session.____('category', 'book')
print(session.cookies()['____'])
~~~

---

## 문제 8 — 필터에 맞는 상품만 선택

쿠키에 저장된 category와 같은 상품 카드만 고른다.

**기대 결과 형태**: 선택된 상품 개수와 첫 상품명이 출력된다.

**빈칸 힌트**: card의 data-category와 filter_category 쿠키를 비교한다.

~~~python
selected = [card for card in cards if card['____'] == session.cookies()['____']]
print(len(selected))
print(selected[0].select_one('____').text.strip())
~~~

---

## 문제 9 — 장바구니에 상품 담기

필터링된 첫 상품을 장바구니에 2개 담는다.

**기대 결과 형태**: cart 리스트와 cart_count가 출력된다.

**빈칸 힌트**: add_to_cart에는 product_id와 quantity를 넘긴다.

~~~python
session.____(selected[0]['____'], quantity=____)
print(session.cart)
print(session.____())
~~~

---

## 문제 10 — 장바구니 금액 계산

카탈로그 가격표를 만들고 장바구니 총액을 계산한다.

**기대 결과 형태**: 상품별 라인과 총액이 출력된다.

**빈칸 힌트**: data-price는 int로 변환해 가격표에 저장한다.

~~~python
price_by_id = {card['data-product-id']: int(card['____']) for card in cards}
name_by_id = {card['data-product-id']: card.select_one('____').text.strip() for card in cards}
total = sum(price_by_id[item['____']] * item['____'] for item in session.cart)
print(total)
~~~

---

## 문제 11 — 쿠키 정책 문장 읽기

cookie_policy.txt를 줄 단위로 읽어 빈 줄을 제외한다.

**기대 결과 형태**: 정책 문장 리스트 또는 줄 출력이 나온다.

**빈칸 힌트**: splitlines와 strip을 사용한다.

~~~python
policy_lines = [line.strip() for line in load_text('____').splitlines() if line.____()]
print(policy_lines)
~~~

---

## 문제 12 — 세션 이벤트 CSV 읽기

session_events.csv를 읽고 이벤트별 개수를 센다.

**기대 결과 형태**: 이벤트 rows 일부와 event counts가 출력된다.

**빈칸 힌트**: read_csv_rows와 Counter를 사용한다.

~~~python
events = ____(____)
print(events[:3])
print(Counter(row['____'] for row in events))
~~~

---

## 문제 13 — 이벤트 로그 재생하기

CSV 이벤트를 DemoSession에 순서대로 적용해 장바구니 상태를 만든다.

**기대 결과 형태**: 재생 후 cart와 cart_count가 출력된다.

**빈칸 힌트**: add와 remove 이벤트만 장바구니에 반영한다.

~~~python
replay = DemoSession()
replay.login('replay-user', token)
for row in events:
    if row['event'] == 'add':
        replay.____(row['product_id'], int(row['quantity']))
    elif row['event'] == 'remove':
        replay.____(row['product_id'], int(row['quantity']))
print(replay.cart)
print(replay.____())
~~~

---

## 문제 14 — 세션 로그 CSV 저장하기

session.log를 outputs/lesson05/session_log.csv로 저장한다.

**기대 결과 형태**: 저장 경로와 로그 행 수가 출력된다.

**빈칸 힌트**: DictWriter fieldnames는 로그 key 전체에서 만든다.

~~~python
output_dir = Path('outputs/lesson05')
output_dir.mkdir(parents=True, exist_ok=True)
log_path = output_dir / 'session_log.csv'
fieldnames = sorted({key for row in session.log for key in row.keys()})
with log_path.open('w', newline='', encoding='utf-8') as f:
    writer = csv.DictWriter(f, fieldnames=____)
    writer.____()
    writer.writerows(session.log)
print(log_path, len(session.log))
~~~

---

## 문제 15 — 상태 요약 JSON 저장하기

로그인 여부, 쿠키 key, cart_count, 정책 문장 수를 JSON으로 저장한다.

**기대 결과 형태**: 요약 딕셔너리와 저장 경로가 출력된다.

**빈칸 힌트**: 쿠키 값 전체가 아니라 key 목록만 저장한다.

~~~python
summary = {'logged_in': session.____(), 'cookie_keys': sorted(session.cookies().____()), 'cart_count': session.____(), 'policy_count': len(policy_lines)}
summary_path = output_dir / 'session_summary.json'
summary_path.write_text(json.dumps(summary, ensure_ascii=False, indent=2), encoding='utf-8')
print(____)
print(summary_path)
~~~

---

## 문제 풀이 기준

| 구간 | 목표 | 확인할 것 |
|---|---|---|
| 문제 1~4 | 로그인 폼과 세션 상태 | h1, csrf_token, input name, 쿠키 |
| 문제 5~8 | 카탈로그와 필터 | product-card, data-category, filter_category |
| 문제 9~10 | 장바구니 상태 | product_id, quantity, total |
| 문제 11~13 | 정책과 이벤트 | cookie_policy, session_events, replay |
| 문제 14~15 | 산출물 저장 | session_log.csv, session_summary.json |

세션 문제는 실행 순서가 중요하다. session을 만들기 전에 session.set_filter를 실행할 수 없고, token을 읽기 전에 login을 호출할 수 없다. 오류가 나면 셀을 부분 실행하지 말고 환경 셀부터 순서대로 다시 실행한다.

## 오류가 났을 때 확인 순서

1. NameError가 나면 이전 셀을 실행했는지 확인한다.
2. ValueError가 나면 token 값이 safe-token-05인지 확인한다.
3. KeyError가 나면 쿠키 key 또는 카드 data 속성 이름을 확인한다.
4. cart_count가 예상과 다르면 add/remove 이벤트 순서를 확인한다.
5. JSON 저장 오류가 나면 summary 안에 직렬화할 수 없는 객체가 있는지 확인한다.

## 제출 전 자체 점검

- 실제 계정, 실제 쿠키, 비밀번호를 코드나 로그에 넣지 않았는가.
- outputs/lesson05/session_log.csv가 생성되는가.
- outputs/lesson05/session_summary.json이 생성되는가.
- JSON에는 쿠키 값 전체가 아니라 cookie_keys만 저장되는가.
- 이벤트 로그를 재생했을 때 장바구니 수량이 설명 가능한가.
- 새 런타임에서 전체 실행해도 같은 결과가 나오는가.

## 빈칸 난이도 안내

| 문제 | 난이도 | 막혔을 때 확인할 곳 |
|---:|---|---|
| 1~3 | 기본 | login_form.html의 h1, input, name 속성 |
| 4~8 | 중간 | DemoSession 메서드와 catalog product-card 구조 |
| 9~10 | 중간 | cart 리스트, data-price, product_id |
| 11~13 | 중간+ | cookie_policy.txt, session_events.csv, add/remove 순서 |
| 14~15 | 종합 | DictWriter, JSON 저장, cookie_keys 안전 요약 |

세션 문제는 빈칸 하나가 맞아도 이전 상태가 없으면 실행되지 않는다. 특히 session, token, cards, policy_lines, output_dir 같은 변수는 앞 문제에서 만들어진다. 중간부터 실행했다면 위쪽 셀을 다시 실행한다.

## 상태 관리 안전선

이번 레슨에서는 실제 사이트 로그인, 실제 브라우저 쿠키 복사, 비밀번호 저장을 하지 않는다. DemoSession은 세션 개념을 설명하기 위한 모형이다. 코드가 잘 실행되더라도 실제 서비스의 쿠키를 가져오거나 저장하도록 바꾸면 수업 기준을 벗어난다.

산출물에는 쿠키 key 목록만 남긴다. cookie value, token 원문, session_id 원문, 비밀번호는 저장하지 않는다. 학생이 디버깅을 위해 화면에 잠깐 출력하는 것도 실제 서비스에서는 조심해야 한다.

## 산출물 이름 기준

최종적으로 만드는 파일 이름은 문제와 정확히 맞춘다. CSV 로그는 outputs/lesson05/session_log.csv, JSON 요약은 outputs/lesson05/session_summary.json이다. 이름이 다르면 검수자가 결과를 찾기 어렵고, 다음 자동화 단계에서 파일을 재사용하기도 어렵다.

