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

# 레슨 05 — 세션과 쿠키 상태 관리

[![Open in Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/Kevin-innovation/jupyter-lecture/blob/main/python-web-automation/lectures/05/%5B%ED%95%99%EC%83%9D%EC%9A%A9%5D%20%EB%A0%88%EC%8A%A8%2005%20%E2%80%94%20%EC%84%B8%EC%85%98%EA%B3%BC%20%EC%BF%A0%ED%82%A4%20%EC%83%81%ED%83%9C%20%EA%B4%80%EB%A6%AC.ipynb)

이 레슨은 웹 자동화에서 상태가 이어지는 방식을 다룬다. HTTP 요청은 기본적으로 독립적이지만, 실제 서비스는 로그인 상태, 필터 선택, 장바구니, 최근 본 항목처럼 이전 동작의 영향을 계속 사용한다. 이 흐름을 이해하려면 세션, 쿠키, 이벤트 로그를 함께 봐야 한다.

수업에서는 실제 계정이나 외부 사이트를 사용하지 않는다. login_form.html은 합성 로그인 폼이고, catalog_page.html은 상품 카탈로그 fixture이며, cookie_policy.txt와 session_events.csv는 상태 관리 안전 규칙과 이벤트 흐름을 연습하기 위한 자료다.

## 학습 목표

1. 세션이 요청 사이의 상태를 유지하는 이유를 설명한다.
2. 로그인 폼의 hidden token과 input name을 읽는다.
3. 쿠키로 필터 상태를 저장하고 다시 사용한다.
4. 장바구니 상태를 세션 객체 안에서 관리한다.
5. 세션 이벤트 로그를 CSV/JSON으로 저장하고 안전 규칙을 점검한다.

---

## 1. 환경 준비와 DemoSession

이번 레슨의 DemoSession은 실제 로그인 요청을 보내지 않는다. requests.Session 객체를 내부에 두고, 합성 쿠키와 장바구니 리스트를 통해 상태 변화를 안전하게 관찰한다.

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

실제 서비스의 쿠키를 가져오거나 저장하지 않는다. 수업용 DemoSession은 상태 개념을 배우기 위한 작은 모형이다. 로그인, 필터, 장바구니, 로그가 어떻게 바뀌는지 보는 것이 목적이다.

---

## 2. 로그인 폼에서 token 읽기

login_form.html에는 hidden input으로 csrf_token이 들어 있다. 실제 서비스에서는 이런 token이 요청 위조를 막는 역할을 한다. 수업에서는 token을 읽고 올바른 값인지 확인하는 흐름만 연습한다.

~~~python
form_soup = read_soup('login_form.html')
heading = form_soup.select_one('h1').text.strip()
form = form_soup.select_one('#login-form')
token = form.select_one('input[name="csrf_token"]')['value']
inputs = [tag.get('name') for tag in form.select('input')]
print(heading)
print('action:', form['action'])
print('token:', token)
print('inputs:', inputs)
~~~

hidden token은 화면에 보이지 않지만 요청에는 필요할 수 있다. 자동화에서 hidden 값을 무시하면 폼 제출 흐름을 이해하기 어렵다. 반대로 실제 서비스 token을 저장하거나 공유하는 것은 금지해야 한다.

---

## 3. 세션 로그인 상태 만들기

DemoSession.login은 token이 맞을 때만 쿠키를 설정한다. demo_user와 session_id는 합성 값이다. 로그인 이후 is_logged_in과 cookies를 확인하면 상태가 유지되는지 볼 수 있다.

~~~python
session = DemoSession()
print('before:', session.is_logged_in(), session.cookies())
session.login('student01', token)
print('after:', session.is_logged_in(), session.cookies())
print('log:', session.log)
~~~

세션은 단순히 로그인 함수 하나를 실행하는 것이 아니다. 이후 필터, 장바구니, 로그가 같은 객체 안에서 이어지도록 만드는 컨테이너다. 상태가 예상과 다르면 쿠키 key와 로그 순서를 확인한다.

---

## 4. 카탈로그 카드 읽기

catalog_page.html에는 article.product-card가 10개 있다. 각 카드에는 data-product-id, data-category, data-price가 들어 있고, 화면에는 상품명과 가격이 보인다.

~~~python
catalog_soup = read_soup('catalog_page.html')
cards = catalog_soup.select('article.product-card')
products = []
for card in cards:
    products.append({
        'product_id': card['data-product-id'],
        'category': card['data-category'],
        'price': int(card['data-price']),
        'name': card.select_one('.name').text.strip(),
        'price_text': card.select_one('.price').text.strip(),
    })
print('products:', len(products))
print(products[:3])
~~~

가격은 data-price 속성의 숫자를 쓰는 편이 안정적이다. 화면의 "18,000원" 문자열은 사람이 보기 좋지만 계산 전에는 쉼표와 원 문자를 제거해야 한다.

---

## 5. 쿠키로 필터 상태 유지하기

카테고리 필터는 사용자가 선택한 화면 상태다. 여기서는 filter_category 쿠키에 book, kit, tool 같은 값을 저장하고, 해당 카테고리 상품만 골라낸다.

~~~python
session.set_filter('category', 'book')
selected_category = session.cookies()['filter_category']
book_products = [product for product in products if product['category'] == selected_category]
print('cookie:', session.cookies())
print('selected:', len(book_products))
print([product['name'] for product in book_products])
~~~

쿠키는 자동화에서 편리하지만 민감한 값도 들어갈 수 있다. 이 수업에서는 합성 필터 값만 쿠키로 다루고, 비밀번호나 실제 인증 토큰은 절대 쿠키 저장 예제로 쓰지 않는다.

---

## 6. 장바구니 상태 관리

장바구니는 세션 상태를 설명하기 좋은 예시다. 상품을 담으면 cart 리스트와 log가 함께 바뀐다. 수량을 더하고 빼며 상태 변화를 확인한다.

~~~python
session.add_to_cart(book_products[0]['product_id'], quantity=2)
session.add_to_cart('kit-1', quantity=1)
print('cart:', session.cart)
print('cart count:', session.cart_count())
print('last log:', session.log[-2:])
~~~

상태 변화 함수는 결과만 바꾸지 말고 로그도 남겨야 한다. 나중에 어떤 순서로 상태가 바뀌었는지 확인할 수 있어야 디버깅이 가능하다.

---

## 7. 장바구니 금액 계산

상품 가격은 catalog에서 읽은 products에 들어 있다. product_id를 key로 가격표를 만들면 cart와 연결해 총액을 계산할 수 있다.

~~~python
price_by_id = {product['product_id']: product['price'] for product in products}
name_by_id = {product['product_id']: product['name'] for product in products}
cart_total = sum(price_by_id[item['product_id']] * item['quantity'] for item in session.cart)
cart_lines = [f"{name_by_id[item['product_id']]} x {item['quantity']}" for item in session.cart]
print(cart_lines)
print('total:', cart_total)
~~~

상태 객체 안에는 상품 id와 수량만 두고, 이름과 가격은 catalog 데이터에서 조회하는 구조가 깔끔하다. 같은 정보를 여러 곳에 복사하면 값이 어긋날 수 있다.

---

## 8. 쿠키 정책 읽기

cookie_policy.txt에는 이 수업에서 지켜야 할 쿠키 안전 규칙이 들어 있다. 자동화 코드도 중요하지만 어떤 값을 저장하지 말아야 하는지 기준을 세우는 것이 더 중요하다.

~~~python
policy_lines = [line.strip() for line in load_text('cookie_policy.txt').splitlines() if line.strip()]
for line in policy_lines:
    print('-', line)
~~~

수업 기준은 분명하다. 실제 개인정보를 쿠키 예제로 쓰지 않고, 비밀번호를 쿠키에 저장하지 않고, 수업이 끝나면 세션 상태를 초기화한다.

---

## 9. 세션 이벤트 로그 읽기

session_events.csv는 사용자의 합성 이벤트 흐름이다. login, filter, add, remove, checkout, logout이 시간 순서로 들어 있다. 이 파일을 읽으면 세션 상태를 재생할 수 있다.

~~~python
events = read_csv_rows('session_events.csv')
print(events[:4])
print('event counts:', Counter(row['event'] for row in events))
~~~

이벤트 로그는 실제 자동화 디버깅에서 매우 중요하다. 화면만 보면 마지막 상태만 보이지만 로그를 보면 어떤 순서로 상태가 바뀌었는지 알 수 있다.

---

## 10. 이벤트를 재생해 상태 만들기

CSV의 이벤트를 순서대로 읽어 DemoSession에 적용한다. filter 이벤트는 쿠키를 바꾸고, add/remove 이벤트는 장바구니를 바꾼다.

~~~python
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
print('cookies:', replay.cookies())
print('cart:', replay.cart)
print('cart count:', replay.cart_count())
~~~

이벤트 재생은 상태 관리의 핵심 훈련이다. 최종 결과만 맞히는 것이 아니라, 어떤 이벤트가 어떤 상태를 바꿨는지 추적할 수 있어야 한다.

---

## 11. 세션 로그 저장하기

세션 상태를 바꾸는 함수는 log를 남긴다. 이 log를 CSV로 저장하면 수업 중 어떤 동작이 발생했는지 검토할 수 있다.

~~~python
output_dir = Path('outputs/lesson05')
output_dir.mkdir(parents=True, exist_ok=True)
log_path = output_dir / 'session_log.csv'
fieldnames = sorted({key for row in session.log for key in row.keys()})
with log_path.open('w', newline='', encoding='utf-8') as f:
    writer = csv.DictWriter(f, fieldnames=fieldnames)
    writer.writeheader()
    writer.writerows(session.log)
print(log_path)
print(log_path.read_text(encoding='utf-8'))
~~~

로그에는 민감정보를 넣지 않는 것이 원칙이다. 이 수업에서도 session_id 원문이나 token 값을 로그에 저장하지 않는다. 이벤트 이름과 합성 사용자명, 상품 id 정도만 남긴다.

---

## 12. 상태 요약 JSON 만들기

CSV는 행 단위 로그에 좋고, JSON은 현재 상태 요약에 좋다. 쿠키 key 목록, 장바구니 수량, 총액, 이벤트 수를 JSON으로 저장해 보자.

~~~python
state_summary = {
    'logged_in': session.is_logged_in(),
    'cookie_keys': sorted(session.cookies().keys()),
    'cart_count': session.cart_count(),
    'cart_total': cart_total,
    'event_count': len(session.log),
    'policy_count': len(policy_lines),
}
summary_path = output_dir / 'session_summary.json'
summary_path.write_text(json.dumps(state_summary, ensure_ascii=False, indent=2), encoding='utf-8')
print(state_summary)
print(summary_path)
~~~

요약 JSON에도 쿠키 값 전체를 넣지 않고 key만 저장한다. 자동화 산출물에는 필요한 정보만 남긴다는 습관이 중요하다.

---

## 13. 상태 초기화 확인

새 DemoSession을 만들면 쿠키와 장바구니가 비어 있어야 한다. 이 점을 확인하면 세션 객체가 상태를 어디에 들고 있는지 이해할 수 있다.

~~~python
fresh = DemoSession()
print('fresh logged in:', fresh.is_logged_in())
print('fresh cookies:', fresh.cookies())
print('fresh cart:', fresh.cart)
print('fresh log:', fresh.log)
~~~

세션 상태는 전역으로 자동 유지되는 것이 아니다. 같은 객체를 계속 써야 이어지고, 새 객체를 만들면 초기화된다. 실제 자동화에서도 세션 객체를 어디에서 만들고 어디까지 전달하는지가 중요하다.

---

## 14. 디버깅 루틴

세션 결과가 예상과 다르면 다음 순서로 본다. token이 맞는지, is_logged_in이 True인지, 쿠키 key가 있는지, 필터 값이 상품 category와 같은지, 장바구니 product_id가 catalog에 존재하는지 확인한다.

~~~python
debug_report = {
    'token_ok': token == 'safe-token-05',
    'logged_in': session.is_logged_in(),
    'has_filter': 'filter_category' in session.cookies(),
    'cart_ids_known': all(item['product_id'] in price_by_id for item in session.cart),
    'log_events': [row['event'] for row in session.log],
}
print(debug_report)
~~~

디버깅은 print를 많이 넣는 것이 아니라 확인 지점을 정하는 것이다. 상태 관리 코드에서는 순서와 key가 중요하므로 체크리스트를 만들어 두면 오류를 빠르게 좁힐 수 있다.

---

## 15. 실제 사이트 확장 전 안전 기준

실제 사이트에 세션 자동화를 적용할 때는 더 엄격해야 한다. 자동 로그인, 쿠키 저장, 인증 토큰 재사용은 서비스 약관과 보안 정책을 위반할 수 있다. 수업에서는 절대 실제 계정 쿠키를 저장하지 않는다.

| 기준 | 수업 원칙 |
|---|---|
| 로그인 | 실제 계정 요청을 보내지 않는다 |
| 쿠키 | 합성 쿠키 key만 다룬다 |
| 비밀번호 | 코드와 로그에 저장하지 않는다 |
| 토큰 | 원문을 제출 산출물에 남기지 않는다 |
| 로그 | 상태 변화와 합성 상품 id만 저장한다 |

이 레슨의 목적은 인증 우회가 아니라 상태 추적이다. 상태가 어디에 저장되고, 어떤 이벤트로 바뀌고, 무엇을 로그에 남기면 안 되는지 구분하는 것이 핵심이다.

---

## 16. 쿠키 값과 쿠키 key를 분리해서 보기

상태 요약을 만들 때 가장 중요한 안전 기준은 쿠키 값 전체를 저장하지 않는 것이다. 쿠키 key는 어떤 상태가 존재하는지 설명하는 데 쓸 수 있지만, 쿠키 value는 인증이나 개인 상태를 담을 수 있다. 이번 수업의 값은 합성이지만 실제 사이트로 확장할 때는 같은 습관을 지켜야 한다.

~~~python
cookie_snapshot = session.cookies()
safe_cookie_view = {
    'cookie_keys': sorted(cookie_snapshot.keys()),
    'cookie_count': len(cookie_snapshot),
    'has_session': 'session_id' in cookie_snapshot,
    'has_filter': 'filter_category' in cookie_snapshot,
}
print(safe_cookie_view)
~~~

학생에게 쿠키 딕셔너리를 그대로 저장하지 말고 key 목록이나 개수만 남기게 한다. 디버깅에는 충분하고, 민감 상태를 산출물로 남길 위험은 줄어든다. 실제 운영 코드에서도 이 원칙은 로그 설계의 기본이다.

---

## 17. 이벤트 로그와 현재 상태의 차이

session.log는 현재 세션 객체에서 직접 실행한 동작의 기록이고, session_events.csv는 외부에서 주어진 합성 이벤트 기록이다. 둘을 섞으면 어떤 상태가 실제 조작 결과인지, 어떤 상태가 재생 결과인지 알기 어렵다. 그래서 변수 이름을 session과 replay로 분리했다.

~~~python
session_event_names = [row['event'] for row in session.log]
replay_event_names = [row['event'] for row in replay.log]
print('session events:', session_event_names)
print('replay events:', replay_event_names[:8])
print('session cart:', session.cart_count())
print('replay cart:', replay.cart_count())
~~~

운영 자동화에서 로그는 두 종류가 있다. 하나는 내가 이번 실행에서 만든 작업 로그이고, 다른 하나는 외부 시스템에서 내려받은 이벤트 로그다. 둘을 구분해야 오류 원인을 제대로 찾을 수 있다.

---

## 18. 상태 관리 코드의 성공 조건

세션 자동화는 화면이 열렸다고 성공이 아니다. 상태가 기대한 순서로 변하고, 민감정보를 저장하지 않고, 산출물이 재실행 가능한 형태로 남아야 한다. 이번 레슨에서는 다음 조건을 만족하면 기본 성공으로 본다.

| 조건 | 확인 코드 | 이유 |
|---|---|---|
| 로그인 상태 | session.is_logged_in() | 세션 쿠키 생성 확인 |
| 필터 상태 | filter_category in cookies | 화면 선택 상태 유지 확인 |
| 장바구니 상태 | session.cart_count() > 0 | 상태 변화 확인 |
| 로그 저장 | session_log.csv 존재 | 재검토 가능성 확인 |
| 안전 요약 | cookie_keys만 저장 | 민감정보 노출 방지 |

~~~python
success_checks = {
    'logged_in': session.is_logged_in(),
    'filter_saved': 'filter_category' in session.cookies(),
    'cart_has_items': session.cart_count() > 0,
    'log_file_exists': Path('outputs/lesson05/session_log.csv').exists(),
    'summary_file_exists': Path('outputs/lesson05/session_summary.json').exists(),
}
print(success_checks)
print('lesson ok:', all(success_checks.values()))
~~~

이 표와 코드는 학생이 자기 답안을 검수할 때 사용할 수 있다. 특히 마지막 두 조건은 산출물 저장 문제를 실행한 뒤에만 True가 된다. 전체 실행 순서가 중요하다는 점을 다시 확인할 수 있다.

