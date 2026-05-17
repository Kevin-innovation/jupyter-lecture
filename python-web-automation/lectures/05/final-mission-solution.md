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

> 🔒 교사용. 학생에게는 최종 미션 문제 파일만 공유한다.

로그인 폼, 카탈로그, 세션 이벤트를 사용해 안전한 세션 상태 리포트를 만든다.

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

## 모범 답안

~~~python
form_html = load_text('login_form.html')
soup = BeautifulSoup(form_html, 'html.parser')
token = soup.select_one('input[name="csrf_token"]')['value']
session = DemoSession()
session.login('student01', token)
session.set_filter('category', 'book')
catalog = BeautifulSoup(load_text('catalog_page.html'), 'html.parser')
cards = catalog.select('article.product-card')
selected = [card for card in cards if card['data-category'] == session.cookies()['filter_category']]
for card in selected[:2]:
    session.add_to_cart(card['data-product-id'], quantity=1)
price_map = {card['data-product-id']: clean_int(card['data-price']) for card in cards}
total = sum(price_map[item['product_id']] * item['quantity'] for item in session.cart)
with open('lesson05_session_report.csv', 'w', newline='', encoding='utf-8') as f:
    writer = csv.DictWriter(f, fieldnames=['event','user','name','value','product_id','quantity'])
    writer.writeheader(); writer.writerows(session.log)
print('logged_in:', session.is_logged_in())
print('cart_count:', session.cart_count())
print('total:', total)
~~~

## 채점 메모

- 코드가 한 번 실행되어 산출물을 만들고, 다시 실행해도 같은 결과가 나와야 한다.
- 상태 변화, selector, 저장 경로, 수집 개수가 요약 문장과 충돌하지 않아야 한다.
- 실제 사이트로 옮길 때 필요한 요청 간격과 오류 처리 언급이 있어야 한다.

### 보강 설명 1

레슨 05 최종 미션 정답은 실행 결과만 맞추는 것이 아니라 재현 가능한 절차를 남기는 것이 핵심이다. 입력 파일, 반복 단위, selector 또는 상태 변수, 저장 경로를 분리하면 오류 위치를 빠르게 찾을 수 있다.

### 보강 설명 2

학생이 막히면 완성 코드를 보여주기보다 HTML 구조, CSV 헤더, 상태 변화 순서를 먼저 말로 설명하게 한다. 구조를 설명할 수 있으면 코드도 안정된다.

### 보강 설명 3

실제 자동화는 다음 주에도 다시 실행되어야 한다. 그래서 쿠키, 폼 입력, selector, wait, 로그 같은 운영 요소를 초반부터 명시적으로 다룬다.

### 보강 설명 4

외부 사이트로 확장할 때는 약관, robots.txt, 요청 간격, 개인정보 여부를 먼저 확인한다. 이 코스의 fixture는 안전한 반복 연습을 위한 합성 데이터다.

### 보강 설명 5

레슨 05 최종 미션 정답은 실행 결과만 맞추는 것이 아니라 재현 가능한 절차를 남기는 것이 핵심이다. 입력 파일, 반복 단위, selector 또는 상태 변수, 저장 경로를 분리하면 오류 위치를 빠르게 찾을 수 있다.

### 보강 설명 6

학생이 막히면 완성 코드를 보여주기보다 HTML 구조, CSV 헤더, 상태 변화 순서를 먼저 말로 설명하게 한다. 구조를 설명할 수 있으면 코드도 안정된다.
