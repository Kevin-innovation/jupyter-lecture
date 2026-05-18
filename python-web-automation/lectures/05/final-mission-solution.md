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

# 레슨 05 — 최종 미션 모범 답안

[![Open in Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/Kevin-innovation/jupyter-lecture/blob/main/python-web-automation/lectures/05/%EB%A0%88%EC%8A%A8%2005%20%E2%80%94%20%EC%84%B8%EC%85%98%EA%B3%BC%20%EC%BF%A0%ED%82%A4%20%EC%83%81%ED%83%9C%20%EA%B4%80%EB%A6%AC.ipynb)

로그인 폼, 카탈로그, 쿠키 정책, 세션 이벤트 CSV를 사용해 안전한 세션 상태 요약을 만든다.

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

## 모범 답안

~~~python
output_dir = Path('outputs/lesson05/final')
output_dir.mkdir(parents=True, exist_ok=True)

form_soup = read_soup('login_form.html')
token = form_soup.select_one('input[name="csrf_token"]')['value']
input_names = [tag.get('name') for tag in form_soup.select('input')]

catalog_soup = read_soup('catalog_page.html')
products = []
for card in catalog_soup.select('article.product-card'):
    products.append({
        'product_id': card['data-product-id'],
        'category': card['data-category'],
        'price': int(card['data-price']),
        'name': card.select_one('.name').text.strip(),
    })
price_by_id = {product['product_id']: product['price'] for product in products}

session = DemoSession()
session.login('final-user', token)
session.set_filter('category', 'book')
for product in products:
    if product['category'] == session.cookies()['filter_category']:
        session.add_to_cart(product['product_id'], 1)
        break
session.add_to_cart('kit-1', 1)

policy_lines = [line.strip() for line in load_text('cookie_policy.txt').splitlines() if line.strip()]
events = read_csv_rows('session_events.csv')
replay = DemoSession()
replay.login('replay-user', token)
for row in events:
    if row['event'] == 'filter':
        replay.set_filter('category', row['product_id'])
    elif row['event'] == 'add':
        replay.add_to_cart(row['product_id'], int(row['quantity']))
    elif row['event'] == 'remove':
        replay.remove_from_cart(row['product_id'], int(row['quantity']))
    elif row['event'] == 'logout':
        replay.logout()

log_path = output_dir / 'session_log.csv'
fieldnames = sorted({key for row in session.log for key in row.keys()})
with log_path.open('w', newline='', encoding='utf-8') as f:
    writer = csv.DictWriter(f, fieldnames=fieldnames)
    writer.writeheader()
    writer.writerows(session.log)

cart_total = sum(price_by_id[item['product_id']] * item['quantity'] for item in session.cart)
summary = {
    'input_names': input_names,
    'logged_in': session.is_logged_in(),
    'cookie_keys': sorted(session.cookies().keys()),
    'cart_count': session.cart_count(),
    'cart_total': cart_total,
    'session_event_count': len(session.log),
    'replay_cart_count': replay.cart_count(),
    'policy_count': len(policy_lines),
    'event_counts': dict(Counter(row['event'] for row in events)),
}
summary_path = output_dir / 'session_summary.json'
summary_path.write_text(json.dumps(summary, ensure_ascii=False, indent=2), encoding='utf-8')
print('log:', log_path, len(session.log))
print('summary:', summary)
~~~

## 왜 이 풀이가 기준 답안인지

폼 구조, 카탈로그, 정책, 이벤트 로그를 각각 읽고 DemoSession 상태로 연결했다. CSV에는 합성 이벤트 로그만 저장하고, JSON에는 쿠키 값이 아니라 cookie_keys만 저장한다. session 상태와 replay 상태를 분리해 현재 조작 결과와 이벤트 재생 결과를 따로 검증할 수 있다.

## 채점 메모

- token 원문, session_id 값, 비밀번호가 저장 산출물에 없어야 한다.
- session_log.csv에는 header와 event 행이 있어야 한다.
- session_summary.json에는 cookie_keys, cart_count, event_count 또는 session_event_count가 있어야 한다.
- 이벤트 재생은 add와 remove 순서를 지켜야 한다.
- 실제 사이트 확장 전 권한, 약관, 개인정보, 저장 기준을 설명할 수 있어야 한다.
