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

# 레슨 01 — 웹 자동화 개요 + HTTP + BeautifulSoup 입문

이 노트북은 파이썬 웹 자동화 코스의 첫 레슨이다. 1강에서는 실제 사이트를 무리하게 긁지 않고, 수업용으로 만든 정적 HTML 파일을 코랩에서 읽어 파싱한다. 목표는 '웹 페이지를 요청한다 → HTML을 읽는다 → 태그를 고른다 → 필요한 값을 표처럼 만든다' 흐름을 몸에 익히는 것이다.

## 학습 목표

이 레슨을 마치면 다음을 할 수 있다.

1. 웹 자동화와 웹 크롤링의 차이를 설명한다.
2. HTTP 요청, 응답 코드, HTML 문서의 기본 구조를 설명한다.
3. requests로 HTML 텍스트를 가져오고 오류를 확인한다.
4. BeautifulSoup으로 title, tag, class, attribute를 선택한다.
5. 상품 카드와 공지 목록에서 텍스트와 링크를 추출한다.
6. 가격 문자열에서 숫자만 정리한다.
7. 추출한 데이터를 리스트/딕셔너리/CSV로 저장한다.
8. robots.txt와 요청 간격이 왜 필요한지 설명한다.

---

## 1. 웹 자동화란 무엇인가

웹 자동화는 사람이 브라우저에서 반복하는 일을 코드로 줄이는 것이다. 예를 들어 공지 제목을 매주 확인하거나, 쇼핑몰 상품 가격을 표로 정리하거나, 여러 페이지의 파일 링크를 모으는 작업이 있다.

웹 자동화는 크게 두 갈래다.

- HTTP 기반 자동화: requests로 HTML이나 JSON을 직접 받아 처리한다. 빠르고 단순하다.
- 브라우저 기반 자동화: Selenium/Playwright로 실제 브라우저를 움직인다. 클릭, 입력, 동적 렌더링 확인이 가능하지만 느리고 깨지기 쉽다.

1강은 HTTP 기반 자동화다. 화면을 클릭하지 않고 HTML 문서를 직접 읽는다.

### 수업에서 지키는 안전 규칙

- 로그인 우회, 유료 데이터 무단 수집, 개인정보 수집은 하지 않는다.
- 공개 데이터라도 요청을 빠르게 반복하지 않는다.
- 사이트 약관과 robots.txt를 확인한다.
- 수업 중에는 합성 데이터로 구조를 먼저 익힌다.

---

## 2. HTML을 데이터로 보는 법

브라우저 화면은 사람이 보기 좋게 렌더링된 결과다. 파이썬은 화면을 보는 대신 HTML 문자열을 읽는다. HTML은 태그가 중첩된 구조다.

~~~html
<article class="product-card" data-category="Laptop">
  <h2 class="name">D-Lab Air 13</h2>
  <span class="price">1,290,000원</span>
  <a class="detail" href="/products/air-13">상세 보기</a>
</article>
~~~

여기서 파이썬이 고를 수 있는 단서는 다음과 같다.

- 태그 이름: article, h2, span, a
- class: product-card, name, price, detail
- 속성: data-category, href
- 텍스트: D-Lab Air 13, 1,290,000원

BeautifulSoup은 이 구조를 탐색하기 쉽게 바꿔주는 도구다.

---

## 3. 환경 준비

아래 셀은 모든 웹자동화 레슨의 기본 시작 셀이다. 코랩이면 raw GitHub에서 데이터를 읽고, 로컬이면 ./data 폴더에서 읽는다.

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

## 4. HTML 파일 읽기

첫 샘플은 mini_shop.html이다. 실제 쇼핑몰이 아니라 수업용으로 만든 합성 HTML이다.

~~~python
html = load_text('mini_shop.html')
print(type(html))
print(len(html))
print(html[:300])
~~~

HTML은 그냥 긴 문자열이다. 문자열 상태에서는 태그를 찾기 어렵기 때문에 BeautifulSoup 객체로 바꾼다.

~~~python
soup = BeautifulSoup(html, 'html.parser')
print(soup.title.text.strip())
print(soup.select_one('h1').text.strip())
~~~

---

## 5. CSS selector로 원하는 요소 고르기

웹 자동화에서 가장 많이 쓰는 문법은 CSS selector다.

- 'h1': h1 태그 선택
- '.product-card': class가 product-card인 요소 선택
- '#main': id가 main인 요소 선택
- 'article.product-card': article 태그이면서 product-card 클래스인 요소 선택
- '.product-card .name': product-card 안쪽의 name 클래스 선택

~~~python
cards = soup.select('.product-card')
print('상품 카드 수:', len(cards))

first = cards[0]
print(first.select_one('.name').text.strip())
print(first.select_one('.price').text.strip())
print(first['data-category'])
~~~

---

## 6. 텍스트 정리와 숫자 변환

웹에서 가져온 값은 대부분 문자열이다. 가격처럼 쉼표와 원 문자가 섞인 문자열은 숫자로 바꿔야 비교와 정렬이 가능하다.

~~~python
price_text = first.select_one('.price').text.strip()
price = clean_int(price_text)
print(price_text, '->', price)
~~~

---

## 7. 여러 상품을 표 형태로 만들기

반복문으로 각 상품 카드에서 같은 위치의 값을 꺼낸다. 리스트 안에 딕셔너리를 쌓으면 표처럼 다루기 쉽다.

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
print('총 상품:', len(products))
~~~

---

## 8. 조건으로 걸러 보기

리스트 컴프리헨션을 쓰면 원하는 조건의 데이터만 뽑을 수 있다.

~~~python
expensive = [p for p in products if p['price'] >= 1500000]
high_rating = [p for p in products if p['rating'] >= 4.7]
out_of_stock = [p for p in products if p['stock'] == 0]

print('150만원 이상:', len(expensive))
print('평점 4.7 이상:', len(high_rating))
print('품절:', len(out_of_stock))
~~~

---

## 9. 공지 페이지 파싱

두 번째 샘플은 notices.html이다. 공지 목록처럼 행이 반복되는 구조를 연습한다.

~~~python
notice_html = load_text('notices.html')
notice_soup = BeautifulSoup(notice_html, 'html.parser')
rows = notice_soup.select('li.notice-item')
print('공지 수:', len(rows))

first_notice = rows[0]
print(first_notice.select_one('.date').text.strip())
print(first_notice.select_one('.title').text.strip())
print(first_notice.select_one('.department').text.strip())
~~~

---

## 10. CSV로 저장하기

수집 결과는 화면에 print만 하지 말고 파일로 남겨야 한다. 이번 레슨에서는 csv 모듈을 쓴다.

~~~python
output_path = 'lesson01_products.csv'
with open(output_path, 'w', newline='', encoding='utf-8') as f:
    writer = csv.DictWriter(f, fieldnames=['name', 'category', 'price', 'rating', 'stock', 'detail_url'])
    writer.writeheader()
    writer.writerows(products)

print('saved:', output_path)
~~~

---

## 11. 현업 활용 사례

웹 자동화는 데이터 분석보다 먼저 쓰이는 경우가 많다. 분석할 데이터가 파일로 주어지지 않을 때, 웹 페이지의 공지·가격·리뷰·채용공고·자료실 링크를 정리해 데이터셋을 만든다.

예를 들어 가격 비교 서비스는 상품명, 가격, 재고, 판매처 링크를 주기적으로 확인한다. 다만 실서비스는 단순 requests만 쓰지 않는다. robots.txt, 약관, 요청 제한, IP 차단 방지, 중복 저장 방지, 변경 이력 관리까지 운영 규칙이 필요하다.

1강에서는 운영 전체가 아니라 '한 페이지를 안전하게 읽고 표로 바꾸는 기본기'만 익힌다.

## 12. HTTP 응답을 안전하게 확인하는 법

웹 자동화에서 가장 흔한 실수는 "HTML이 왔겠지"라고 가정하고 바로 파싱하는 것이다. 실제 사이트에서는 URL 오타, 권한 없음, 서버 오류, 너무 빠른 요청 때문에 HTML 대신 에러 페이지가 올 수 있다. 그래서 요청 결과를 파싱하기 전에 응답 상태를 확인해야 한다.

> **🛡️ 웹 자동화 안전 한스푼 — HTTP 요청/응답과 status_code**
>
> - **뜻**: 요청은 클라이언트가 서버에 보내는 질문이고, 응답은 서버가 돌려주는 결과다. `status_code` 는 그 결과의 상태 번호다.
> - **왜 중요한가**: 200이 아니면 원하는 HTML이 아닐 수 있다. 404는 주소 없음, 403은 거부, 429는 너무 많은 요청을 뜻할 수 있다.
> - **수업 기준**: 외부 요청 코드는 항상 `timeout` 과 상태 확인을 같이 둔다.
> - **실수 예시**: `requests.get(url).text` 만 쓰고 상태를 확인하지 않은 채 BeautifulSoup에 넘긴다.

~~~python
sample_url = DATA_BASE + '/mini_shop.html' if DATA_BASE.startswith('http') else None

if sample_url:
    response = requests.get(sample_url, timeout=10, headers={'User-Agent': 'D-Lab-Lesson/1.0'})
    print('status:', response.status_code)
    response.raise_for_status()
    print(response.text[:80])
else:
    print('로컬 실행 중이므로 외부 HTTP 요청 예시는 건너뜁니다.')
~~~

> **🛡️ 웹 자동화 안전 한스푼 — timeout**
>
> - **뜻**: 서버가 일정 시간 안에 응답하지 않으면 기다리기를 멈추는 제한 시간이다.
> - **왜 중요한가**: timeout이 없으면 네트워크 문제 하나로 노트북 셀이 계속 멈춰 있을 수 있다.
> - **수업 기준**: `requests.get(url, timeout=10)` 을 기본으로 쓴다.
> - **실수 예시**: 여러 URL을 반복 요청하면서 timeout을 넣지 않아 수업 전체가 멈춘다.

`raise_for_status()` 는 400번대/500번대 응답을 그냥 넘어가지 않게 해 준다. 처음에는 에러가 뜨는 것이 불편해 보이지만, 조용히 틀린 HTML을 파싱하는 것보다 훨씬 안전하다.

---

## 13. selector 디버깅 루틴

selector가 틀리면 BeautifulSoup은 에러를 내지 않고 빈 리스트를 돌려주는 경우가 많다. 그래서 한 번에 긴 코드를 쓰지 말고 다음 순서로 확인한다.

1. 문서 전체가 제대로 읽혔는지 `len(html)` 을 본다.
2. `soup.title` 또는 `h1` 처럼 확실한 요소를 먼저 찾는다.
3. 반복 단위 selector의 개수를 `len(soup.select(...))` 로 본다.
4. 첫 번째 반복 단위 하나에서 내부 요소를 확인한다.
5. 그 다음에야 반복문으로 전체를 처리한다.

~~~python
debug_cards = soup.select('.product-card')
print('cards:', len(debug_cards))

if debug_cards:
    sample = debug_cards[0]
    for selector in ['.name', '.price', '.rating', '.stock', 'a.detail']:
        found = sample.select_one(selector)
        print(selector, '=>', found.text.strip() if found else '없음')
~~~

> **🛡️ 웹 자동화 안전 한스푼 — HTML selector**
>
> - **뜻**: HTML에서 원하는 태그를 고르는 주소 같은 표현이다. `.class`, `#id`, `tag[attr=value]` 를 조합한다.
> - **왜 중요한가**: selector가 불안정하면 사이트 구조가 조금만 바뀌어도 자동화가 깨진다.
> - **수업 기준**: 반복 단위 selector를 먼저 고르고, 내부 selector는 그 안에서 다시 찾는다.
> - **실수 예시**: 전체 문서에서 `.price` 를 한 번만 찾아 첫 상품 가격만 계속 저장한다.

selector가 비어 있으면 HTML을 다시 보는 것이 먼저다. 코드를 더 복잡하게 만들기 전에 "내가 고르려는 반복 단위가 실제로 어떤 태그와 class를 갖는가"를 눈으로 확인한다.

---

## 14. 상대 URL과 절대 URL

HTML 안의 링크는 `/products/air-13` 처럼 상대 경로로 들어 있는 경우가 많다. 사람이 브라우저로 볼 때는 현재 사이트 주소가 자동으로 붙지만, CSV로 저장할 때는 기준 주소를 붙여 절대 URL로 바꿔야 한다.

~~~python
relative_href = first.select_one('a.detail')['href']
absolute_href = urljoin('https://example.com/shop/', relative_href)
print(relative_href)
print(absolute_href)
~~~

상대 URL을 그대로 저장하면 나중에 CSV만 열었을 때 링크가 어디를 가리키는지 알기 어렵다. `urljoin` 은 기준 주소와 상대 경로를 안전하게 합쳐 준다.

> **🛡️ 웹 자동화 안전 한스푼 — relative URL / absolute URL**
>
> - **뜻**: 상대 URL은 현재 사이트를 기준으로 한 짧은 주소이고, 절대 URL은 `https://...` 로 시작하는 완전한 주소다.
> - **왜 중요한가**: 수집 결과를 CSV로 저장하면 기준 사이트 정보가 사라질 수 있다.
> - **수업 기준**: 링크를 저장할 때는 `urljoin` 으로 절대 URL 형태를 만든다.
> - **실수 예시**: `/notice/12` 만 저장해서 나중에 어느 사이트의 공지인지 알 수 없다.

---

## 15. robots.txt와 요청 간격

이번 레슨은 합성 HTML 파일을 쓰기 때문에 실제 서버에 반복 요청하지 않는다. 그래도 처음부터 robots.txt와 요청 간격을 말하는 이유는 습관 때문이다. 자동화 코드는 반복문을 쓰는 순간 사람이 클릭하는 속도보다 훨씬 빨라질 수 있다.

> **🛡️ 웹 자동화 안전 한스푼 — robots.txt**
>
> - **뜻**: 사이트가 자동화 프로그램에게 어느 경로를 허용하거나 제한하는지 알려주는 참고 파일이다.
> - **왜 중요한가**: 공개 페이지라도 운영자가 자동 접근을 제한하고 싶을 수 있다.
> - **수업 기준**: 실제 사이트 예제는 robots와 약관을 확인한 뒤 강사가 지정한 범위에서만 요청한다.
> - **실수 예시**: "브라우저에서 보이니까 마음대로 반복 요청해도 된다"고 생각한다.

> **🛡️ 웹 자동화 안전 한스푼 — 요청 간격(rate limit)**
>
> - **뜻**: 여러 페이지를 요청할 때 요청 사이에 쉬는 시간을 두는 규칙이다.
> - **왜 중요한가**: 너무 빠른 반복 요청은 서버 부하나 차단의 원인이 된다.
> - **수업 기준**: 실제 외부 요청 반복문에는 최소 `time.sleep(1)` 을 둔다.
> - **실수 예시**: 1초에 수십 번씩 URL을 바꿔 요청한다.

~~~python
targets_text = load_text('targets.csv')
print(targets_text.splitlines()[:4])

# 실제 사이트 요청 예시는 아니다. 반복 처리 구조만 보여준다.
for index, filename in enumerate(['mini_shop.html', 'notices.html'], start=1):
    text = load_text(filename)
    print(index, filename, len(text))
    time.sleep(0.2)  # 수업 fixture라 짧게 둔다. 실제 외부 사이트는 더 길게 둔다.
~~~

---

## 16. 실무에서는 무엇을 더 추가하나

실무 자동화는 1강 코드보다 더 많은 보호 장치를 둔다.

| 항목 | 1강에서 하는 것 | 실무에서 추가하는 것 |
|---|---|---|
| 요청 | fixture 또는 raw 파일 읽기 | 재시도, 요청 간격, 실패 로그 |
| 파싱 | selector로 텍스트 추출 | selector 변경 감지, 누락 필드 알림 |
| 저장 | CSV 1개 저장 | 날짜별 파일명, 중복 제거, DB 저장 |
| 검증 | 개수와 일부 값 출력 | 스키마 검증, null 비율, 이전 결과와 비교 |
| 운영 | 노트북 수동 실행 | 스케줄러, 알림, 장애 대응 |

이 표를 보면 1강의 목적이 분명해진다. 오늘은 실무 전체를 다 만들지 않는다. 대신 반복 단위를 고르고, 내부 값을 읽고, 숫자로 바꾸고, CSV로 저장하는 기초 루틴을 정확히 익힌다. 이 루틴이 흔들리면 페이지네이션, 브라우저 자동화, 재시도 같은 뒤 레슨도 모두 흔들린다.

---

## 17. 제출 전 마무리 체크

이번 레슨을 마치기 전에 아래 질문에 답해 본다.

- HTML 문자열을 BeautifulSoup 객체로 바꾸는 이유를 말할 수 있는가?
- `select` 와 `select_one` 의 차이를 말할 수 있는가?
- class selector 앞에 `.` 을 붙이는 이유를 설명할 수 있는가?
- 가격/조회수 문자열을 숫자로 바꾸지 않으면 어떤 문제가 생기는가?
- 상대 URL을 절대 URL로 바꾸는 이유를 말할 수 있는가?
- 실제 사이트에서 같은 코드를 실행하기 전에 robots.txt, 약관, 요청 간격을 확인해야 하는 이유를 말할 수 있는가?

## 데이터 출처와 안전 규칙

이 레슨의 `mini_shop.html`, `notices.html`, `targets.csv`, `robots_sample.txt` 는 수업용 합성 데이터다. 실존 사이트나 개인 정보를 포함하지 않는다. `requests` 예제는 raw GitHub의 수업 파일 또는 로컬 fixture만 읽도록 설계되어 있다. 학생은 강사가 별도로 허가하지 않은 실제 사이트에 반복 요청을 보내지 않는다.
