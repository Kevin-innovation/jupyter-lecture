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

# 레슨 05 — 최종 미션

[![Open in Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/Kevin-innovation/jupyter-lecture/blob/main/python-web-automation/lectures/05/%5B%ED%95%99%EC%83%9D%EC%9A%A9%5D%20%EB%A0%88%EC%8A%A8%2005%20%E2%80%94%20%EC%84%B8%EC%85%98%EA%B3%BC%20%EC%BF%A0%ED%82%A4%20%EC%83%81%ED%83%9C%20%EA%B4%80%EB%A6%AC.ipynb)

로그인 폼, 카탈로그, 쿠키 정책, 세션 이벤트 CSV를 사용해 안전한 세션 상태 요약을 만든다. 이번 미션은 실제 사이트 요청 없이 합성 fixture와 DemoSession만 사용한다.

## 목표

- login_form.html에서 token과 input name을 읽는다.
- catalog_page.html에서 상품 목록과 가격표를 만든다.
- DemoSession에 로그인, 필터, 장바구니 상태를 반영한다.
- session_events.csv를 재생해 상태 변화를 검증한다.
- session_log.csv와 session_summary.json을 저장한다.

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

## 제출 산출물

1. 실행 가능한 노트북
2. outputs/lesson05/session_log.csv
3. outputs/lesson05/session_summary.json
4. 자동화 결과 요약 3문장
5. 쿠키와 세션 안전 규칙 메모 2개

## 구현 조건

- 실제 계정, 실제 쿠키, 비밀번호를 사용하지 않는다.
- token 원문과 session_id 값을 산출물에 저장하지 않는다.
- JSON 요약에는 cookie_keys, cart_count, event_count, policy_count를 포함한다.
- CSV 로그에는 event와 필요한 합성 값만 남긴다.
- 이벤트 재생 결과와 현재 session 상태를 구분해서 설명한다.

## 스타터 코드

~~~python
# 1. form/catalog/policy/event fixture 읽기
session = DemoSession()

# 2. 로그인, 필터, 장바구니 상태 만들기
session_log = []

# 3. CSV/JSON 산출물 저장
# TODO
~~~

## 자동화 결과 요약 양식

- 사용한 fixture:
- 최종 상태:
- 다음 실행 때 조심할 점:

## 안전 규칙 메모 양식

- 쿠키와 token 저장 기준:
- 실제 사이트 확장 전 확인할 점:

## 평가 루브릭

| 항목 | 배점 | 확인 방법 |
|---|---:|---|
| 폼 구조 이해 | 20 | hidden token과 input name을 읽는다 |
| 세션 상태 관리 | 25 | 쿠키, 필터, 장바구니 상태를 설명한다 |
| 이벤트 재생 | 20 | session_events.csv를 순서대로 처리한다 |
| 산출물 저장 | 25 | CSV 로그와 JSON 요약을 만든다 |
| 안전 메모 | 10 | 실제 쿠키와 민감정보를 저장하지 않는다 |

## 제출 전 체크

- 새 런타임에서 전체 실행해도 같은 산출물이 만들어지는가.
- outputs/lesson05 아래에 CSV와 JSON이 생성되는가.
- 쿠키 값 원문, token, 비밀번호가 저장 파일에 들어가지 않았는가.
- 상태 변화 순서를 말로 설명할 수 있는가.
