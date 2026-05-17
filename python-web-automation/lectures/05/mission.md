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

세션과 쿠키 상태 관리 레슨의 학생용 문제 노트북이다. 강의 노트북을 먼저 실행한 뒤 빈칸을 직접 채운다.

## 통과 기준

- 총 15문제 중 12문제 이상 정상 출력이면 통과.
- 문제 1~5는 구조 확인, 6~10은 상태 변화와 반복 처리, 11~15는 로그와 저장이다.
- 정답값은 적지 않는다. 출력 형태와 fixture 구조를 보고 직접 판단한다.

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

## 문제 1 — 로그인 폼 제목 읽기

지시된 값을 코드로 추출하거나 상태를 변경한다.

**기대 결과 형태**: 요구한 값이 한 줄 또는 리스트/딕셔너리 형태로 출력된다.

**빈칸 힌트**: HTML 구조, CSV 헤더, 이전 예제를 확인한 뒤 `____` 부분을 직접 채운다.

~~~python
form_html = ____(____)
soup = ____(form_html, 'html.parser')
print(soup.____('____').text.strip())
~~~

---

## 문제 2 — hidden CSRF 토큰 읽기

지시된 값을 코드로 추출하거나 상태를 변경한다.

**기대 결과 형태**: 요구한 값이 한 줄 또는 리스트/딕셔너리 형태로 출력된다.

**빈칸 힌트**: HTML 구조, CSV 헤더, 이전 예제를 확인한 뒤 `____` 부분을 직접 채운다.

~~~python
token = soup.select_one('input[name="____"]')['____']
print(token)
~~~

---

## 문제 3 — 폼 입력 name 목록 추출

지시된 값을 코드로 추출하거나 상태를 변경한다.

**기대 결과 형태**: 요구한 값이 한 줄 또는 리스트/딕셔너리 형태로 출력된다.

**빈칸 힌트**: HTML 구조, CSV 헤더, 이전 예제를 확인한 뒤 `____` 부분을 직접 채운다.

~~~python
inputs = soup.select('____')
names = [tag.get('____') for tag in inputs]
print(names)
~~~

---

## 문제 4 — DemoSession 생성과 로그인

지시된 값을 코드로 추출하거나 상태를 변경한다.

**기대 결과 형태**: 요구한 값이 한 줄 또는 리스트/딕셔너리 형태로 출력된다.

**빈칸 힌트**: HTML 구조, CSV 헤더, 이전 예제를 확인한 뒤 `____` 부분을 직접 채운다.

~~~python
session = ____()
session.____('student01', token)
print(session.____())
print(session.____())
~~~

---

## 문제 5 — 카탈로그 카드 개수 세기

지시된 값을 코드로 추출하거나 상태를 변경한다.

**기대 결과 형태**: 요구한 값이 한 줄 또는 리스트/딕셔너리 형태로 출력된다.

**빈칸 힌트**: HTML 구조, CSV 헤더, 이전 예제를 확인한 뒤 `____` 부분을 직접 채운다.

~~~python
catalog = BeautifulSoup(load_text('____'), 'html.parser')
cards = catalog.select('____')
print(len(cards))
~~~

---

## 문제 6 — 첫 상품 정보 읽기

지시된 값을 코드로 추출하거나 상태를 변경한다.

**기대 결과 형태**: 요구한 값이 한 줄 또는 리스트/딕셔너리 형태로 출력된다.

**빈칸 힌트**: HTML 구조, CSV 헤더, 이전 예제를 확인한 뒤 `____` 부분을 직접 채운다.

~~~python
first = cards[____]
print(first['____'])
print(first.select_one('____').text.strip())
print(first['____'])
~~~

---

## 문제 7 — 카테고리 필터 쿠키 저장

지시된 값을 코드로 추출하거나 상태를 변경한다.

**기대 결과 형태**: 요구한 값이 한 줄 또는 리스트/딕셔너리 형태로 출력된다.

**빈칸 힌트**: HTML 구조, CSV 헤더, 이전 예제를 확인한 뒤 `____` 부분을 직접 채운다.

~~~python
session.____('category', 'book')
print(session.cookies()['____'])
~~~

---

## 문제 8 — 필터에 맞는 상품만 선택

지시된 값을 코드로 추출하거나 상태를 변경한다.

**기대 결과 형태**: 요구한 값이 한 줄 또는 리스트/딕셔너리 형태로 출력된다.

**빈칸 힌트**: HTML 구조, CSV 헤더, 이전 예제를 확인한 뒤 `____` 부분을 직접 채운다.

~~~python
selected = [card for card in cards if card['____'] == session.cookies()['____']]
print(len(selected))
print(selected[0].select_one('____').text.strip())
~~~

---

## 문제 9 — 장바구니에 상품 담기

지시된 값을 코드로 추출하거나 상태를 변경한다.

**기대 결과 형태**: 요구한 값이 한 줄 또는 리스트/딕셔너리 형태로 출력된다.

**빈칸 힌트**: HTML 구조, CSV 헤더, 이전 예제를 확인한 뒤 `____` 부분을 직접 채운다.

~~~python
session.____(selected[0]['____'], quantity=2)
print(session.____())
print(session.cart)
~~~

---

## 문제 10 — 상품 id로 가격 찾기

지시된 값을 코드로 추출하거나 상태를 변경한다.

**기대 결과 형태**: 요구한 값이 한 줄 또는 리스트/딕셔너리 형태로 출력된다.

**빈칸 힌트**: HTML 구조, CSV 헤더, 이전 예제를 확인한 뒤 `____` 부분을 직접 채운다.

~~~python
price_map = {card['data-product-id']: ____(card['____']) for card in cards}
print(price_map[session.cart[0]['____']])
~~~

---

## 문제 11 — 장바구니 총액 계산

지시된 값을 코드로 추출하거나 상태를 변경한다.

**기대 결과 형태**: 요구한 값이 한 줄 또는 리스트/딕셔너리 형태로 출력된다.

**빈칸 힌트**: HTML 구조, CSV 헤더, 이전 예제를 확인한 뒤 `____` 부분을 직접 채운다.

~~~python
total = 0
for item in session.cart:
    total += price_map[item['____']] * item['____']
print(total)
~~~

---

## 문제 12 — 세션 이벤트 CSV 읽기

지시된 값을 코드로 추출하거나 상태를 변경한다.

**기대 결과 형태**: 요구한 값이 한 줄 또는 리스트/딕셔너리 형태로 출력된다.

**빈칸 힌트**: HTML 구조, CSV 헤더, 이전 예제를 확인한 뒤 `____` 부분을 직접 채운다.

~~~python
events = list(csv.DictReader(load_text('____').splitlines()))
print(len(events))
print(events[0]['____'])
~~~

---

## 문제 13 — add 이벤트만 필터링

지시된 값을 코드로 추출하거나 상태를 변경한다.

**기대 결과 형태**: 요구한 값이 한 줄 또는 리스트/딕셔너리 형태로 출력된다.

**빈칸 힌트**: HTML 구조, CSV 헤더, 이전 예제를 확인한 뒤 `____` 부분을 직접 채운다.

~~~python
adds = [row for row in events if row['____'] == 'add']
print(len(adds))
print([row['product_id'] for row in adds])
~~~

---

## 문제 14 — 쿠키 정책 줄 수 확인

지시된 값을 코드로 추출하거나 상태를 변경한다.

**기대 결과 형태**: 요구한 값이 한 줄 또는 리스트/딕셔너리 형태로 출력된다.

**빈칸 힌트**: HTML 구조, CSV 헤더, 이전 예제를 확인한 뒤 `____` 부분을 직접 채운다.

~~~python
policy = load_text('____')
lines = [line for line in policy.splitlines() if line.strip()]
print(len(____))
print(lines[0])
~~~

---

## 문제 15 — 세션 로그 CSV 저장

지시된 값을 코드로 추출하거나 상태를 변경한다.

**기대 결과 형태**: 요구한 값이 한 줄 또는 리스트/딕셔너리 형태로 출력된다.

**빈칸 힌트**: HTML 구조, CSV 헤더, 이전 예제를 확인한 뒤 `____` 부분을 직접 채운다.

~~~python
with open('lesson05_session_log.csv', 'w', newline='', encoding='utf-8') as f:
    writer = csv.DictWriter(f, fieldnames=['event', 'user', 'name', 'value', 'product_id', 'quantity'])
    writer.____()
    writer.____(session.log)
print('saved:', len(session.log))
~~~

---

### 보강 설명 1

레슨 05 실습 문제은 실행 결과만 맞추는 것이 아니라 재현 가능한 절차를 남기는 것이 핵심이다. 입력 파일, 반복 단위, selector 또는 상태 변수, 저장 경로를 분리하면 오류 위치를 빠르게 찾을 수 있다.

### 보강 설명 2

학생이 막히면 완성 코드를 보여주기보다 HTML 구조, CSV 헤더, 상태 변화 순서를 먼저 말로 설명하게 한다. 구조를 설명할 수 있으면 코드도 안정된다.

### 보강 설명 3

실제 자동화는 다음 주에도 다시 실행되어야 한다. 그래서 쿠키, 폼 입력, selector, wait, 로그 같은 운영 요소를 초반부터 명시적으로 다룬다.

### 보강 설명 4

외부 사이트로 확장할 때는 약관, robots.txt, 요청 간격, 개인정보 여부를 먼저 확인한다. 이 코스의 fixture는 안전한 반복 연습을 위한 합성 데이터다.

### 보강 설명 5

레슨 05 실습 문제은 실행 결과만 맞추는 것이 아니라 재현 가능한 절차를 남기는 것이 핵심이다. 입력 파일, 반복 단위, selector 또는 상태 변수, 저장 경로를 분리하면 오류 위치를 빠르게 찾을 수 있다.

### 보강 설명 6

학생이 막히면 완성 코드를 보여주기보다 HTML 구조, CSV 헤더, 상태 변화 순서를 먼저 말로 설명하게 한다. 구조를 설명할 수 있으면 코드도 안정된다.

### 보강 설명 7

실제 자동화는 다음 주에도 다시 실행되어야 한다. 그래서 쿠키, 폼 입력, selector, wait, 로그 같은 운영 요소를 초반부터 명시적으로 다룬다.

### 보강 설명 8

외부 사이트로 확장할 때는 약관, robots.txt, 요청 간격, 개인정보 여부를 먼저 확인한다. 이 코스의 fixture는 안전한 반복 연습을 위한 합성 데이터다.

### 보강 설명 9

레슨 05 실습 문제은 실행 결과만 맞추는 것이 아니라 재현 가능한 절차를 남기는 것이 핵심이다. 입력 파일, 반복 단위, selector 또는 상태 변수, 저장 경로를 분리하면 오류 위치를 빠르게 찾을 수 있다.

### 보강 설명 10

학생이 막히면 완성 코드를 보여주기보다 HTML 구조, CSV 헤더, 상태 변화 순서를 먼저 말로 설명하게 한다. 구조를 설명할 수 있으면 코드도 안정된다.

### 보강 설명 11

실제 자동화는 다음 주에도 다시 실행되어야 한다. 그래서 쿠키, 폼 입력, selector, wait, 로그 같은 운영 요소를 초반부터 명시적으로 다룬다.

### 보강 설명 12

외부 사이트로 확장할 때는 약관, robots.txt, 요청 간격, 개인정보 여부를 먼저 확인한다. 이 코스의 fixture는 안전한 반복 연습을 위한 합성 데이터다.

### 보강 설명 13

레슨 05 실습 문제은 실행 결과만 맞추는 것이 아니라 재현 가능한 절차를 남기는 것이 핵심이다. 입력 파일, 반복 단위, selector 또는 상태 변수, 저장 경로를 분리하면 오류 위치를 빠르게 찾을 수 있다.

### 보강 설명 14

학생이 막히면 완성 코드를 보여주기보다 HTML 구조, CSV 헤더, 상태 변화 순서를 먼저 말로 설명하게 한다. 구조를 설명할 수 있으면 코드도 안정된다.

### 보강 설명 15

실제 자동화는 다음 주에도 다시 실행되어야 한다. 그래서 쿠키, 폼 입력, selector, wait, 로그 같은 운영 요소를 초반부터 명시적으로 다룬다.

### 보강 설명 16

외부 사이트로 확장할 때는 약관, robots.txt, 요청 간격, 개인정보 여부를 먼저 확인한다. 이 코스의 fixture는 안전한 반복 연습을 위한 합성 데이터다.

### 보강 설명 17

레슨 05 실습 문제은 실행 결과만 맞추는 것이 아니라 재현 가능한 절차를 남기는 것이 핵심이다. 입력 파일, 반복 단위, selector 또는 상태 변수, 저장 경로를 분리하면 오류 위치를 빠르게 찾을 수 있다.

### 보강 설명 18

학생이 막히면 완성 코드를 보여주기보다 HTML 구조, CSV 헤더, 상태 변화 순서를 먼저 말로 설명하게 한다. 구조를 설명할 수 있으면 코드도 안정된다.

### 보강 설명 19

실제 자동화는 다음 주에도 다시 실행되어야 한다. 그래서 쿠키, 폼 입력, selector, wait, 로그 같은 운영 요소를 초반부터 명시적으로 다룬다.

### 보강 설명 20

외부 사이트로 확장할 때는 약관, robots.txt, 요청 간격, 개인정보 여부를 먼저 확인한다. 이 코스의 fixture는 안전한 반복 연습을 위한 합성 데이터다.
