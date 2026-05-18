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

# 레슨 02 — URL 파라미터와 페이지네이션

[![Open in Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/Kevin-innovation/jupyter-lecture/blob/main/python-web-automation/lectures/02/%5B%ED%95%99%EC%83%9D%EC%9A%A9%5D%20%EB%A0%88%EC%8A%A8%2002%20%E2%80%94%20URL%20%ED%8C%8C%EB%9D%BC%EB%AF%B8%ED%84%B0%EC%99%80%20%ED%8E%98%EC%9D%B4%EC%A7%80%EB%84%A4%EC%9D%B4%EC%85%98.ipynb)

이 레슨은 검색 URL을 문자열이 아니라 구조화된 데이터로 읽는 방법을 다룬다. 학생은 query string을 분해하고 다시 조립하며, 여러 장으로 나뉜 검색 결과를 안전하게 순회한다. 모든 예제는 수업용 합성 HTML fixture를 사용하므로 실제 웹사이트에 반복 요청을 보내지 않는다.

## 학습 목표

1. URL을 scheme, domain, path, query string으로 분해한다.
2. parse_qs, urlencode, urljoin으로 검색 조건과 상대 링크를 안전하게 다룬다.
3. 페이지 번호 규칙을 함수로 분리해 반복 수집 코드를 단순하게 만든다.
4. 페이지네이션 영역의 다음 링크와 현재 페이지 데이터를 구분한다.
5. 여러 페이지에서 모은 결과를 필터링, 집계, CSV 저장까지 연결한다.

---

## 1. URL은 문자열이 아니라 구조다

검색 페이지 URL은 길게 보이지만 실제로는 몇 개의 역할로 나뉜다. 예를 들어 https://example.com/library/search?q=python&category=all&page=2 에서 path는 /library/search이고, query string은 q, category, page 같은 검색 조건을 담는다.

자동화 코드를 만들 때 URL 전체를 문자열로 붙이면 실수하기 쉽다. 검색어에 공백이나 한글이 들어가면 직접 붙인 문자열은 깨질 수 있다. 그래서 파이썬에서는 먼저 URL을 분해하고, 조건은 딕셔너리로 관리한 다음 다시 안전하게 조립한다.

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
    DATA_BASE = 'https://raw.githubusercontent.com/Kevin-innovation/jupyter-lecture/main/python-web-automation/lectures/02/data'
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

def load_bytes(filename):
    if DATA_BASE.startswith('http'):
        url = f'{DATA_BASE}/{filename}'
        response = requests.get(url, timeout=10, headers={'User-Agent': 'D-Lab-Lesson/1.0'})
        response.raise_for_status()
        return response.content
    return Path(DATA_BASE, filename).read_bytes()

def clean_int(text):
    return int(re.sub(r'[^0-9]', '', str(text)))

def page_filename(page):
    return f'search_page_{page}.html'

print('colab:', IS_COLAB)
print('data base:', DATA_BASE)

sample_url = 'https://example.com/library/search?q=python&category=all&page=2'
parsed = urlparse(sample_url)
params = parse_qs(parsed.query)
print('path:', parsed.path)
print('query dict:', params)
print('page:', params['page'][0])
~~~

> **웹 자동화 안전 한스푼 — query string**
>
> - **뜻**: URL의 물음표 뒤에 붙는 검색 조건이다.
> - **왜 중요한가**: 검색어, 페이지 번호, 필터 조건이 여기 들어가므로 잘못 조립하면 다른 데이터를 가져온다.
> - **수업 기준**: query string은 직접 문자열로 이어 붙이지 않고 urlencode로 만든다.
> - **실수 예시**: q, page 값을 더하기 연산으로 붙이다가 공백과 숫자 처리를 놓친다.

---

## 2. 검색 조건을 딕셔너리로 관리하기

검색 자동화는 보통 같은 URL에 조건만 바꿔 여러 번 실행한다. 조건을 딕셔너리로 두면 검색어, 카테고리, 페이지 번호를 코드 안에서 명확하게 볼 수 있고, CSV에서 읽은 조건도 그대로 넣을 수 있다.

urlencode는 공백과 특수문자를 URL에 맞게 인코딩한다. 학생이 직접 퍼센트 인코딩을 외울 필요가 없다.

~~~python
base_url = 'https://example.com/library/search'
query = {'q': 'python automation', 'category': 'course', 'page': 1}
url = base_url + '?' + urlencode(query)
print(url)

query['page'] = 2
print(base_url + '?' + urlencode(query))
~~~

---

## 3. fixture HTML 읽기

이번 레슨의 검색 결과는 search_page_1.html, search_page_2.html, search_page_3.html 세 파일로 준비되어 있다. 각 파일에는 article.result-card 9개가 들어 있고, 각 카드에는 제목, 상세 링크, 카테고리, 날짜, 조회수가 있다.

실제 사이트를 요청하지 않고 파일 fixture를 쓰는 이유는 수업 중 같은 결과를 안정적으로 재현하기 위해서다. 학생이 반복 실행해도 외부 서버에 부하가 가지 않고, 선생님은 selector 오류와 코드 오류를 구분해서 지도할 수 있다.

~~~python
html_text = load_text('search_page_1.html')
soup = BeautifulSoup(html_text, 'html.parser')
heading = soup.select_one('h1').text.strip()
cards = soup.select('article.result-card')
first = cards[0]
print(heading)
print('card count:', len(cards))
print(first.select_one('.title a').text.strip())
print(first['data-category'], first['data-page'])
~~~

> **웹 자동화 안전 한스푼 — fixture**
>
> - **뜻**: 수업이나 테스트를 위해 고정해 둔 샘플 데이터다.
> - **왜 중요한가**: 외부 사이트 상태가 바뀌어도 수업 결과가 흔들리지 않는다.
> - **수업 기준**: 1~5강은 실제 사이트 대신 합성 fixture로 selector와 반복 구조를 연습한다.
> - **실수 예시**: fixture에서 충분히 연습하지 않고 실제 사이트를 빠르게 반복 요청한다.

---

## 4. 카드 하나를 레코드로 바꾸기

HTML 카드 하나를 그대로 저장하면 나중에 정렬하거나 필터링하기 어렵다. 자동화 결과는 사람이 읽는 화면에서 파이썬이 다루기 쉬운 딕셔너리로 바꾸는 과정이 필요하다.

여기서는 제목, 카테고리, 페이지 번호, 날짜, 조회수, 상세 URL을 하나의 딕셔너리로 만든다. 조회수는 조회 1,023 같은 문자열이므로 숫자만 남겨 정수로 바꾼다. 링크는 /library/... 형태의 상대 경로이므로 urljoin으로 절대 URL을 만든다.

~~~python
def parse_result_card(card, base='https://example.com'):
    link = card.select_one('.title a')
    return {
        'title': link.text.strip(),
        'category': card['data-category'],
        'page': int(card['data-page']),
        'rank': int(card['data-rank']),
        'date': card.select_one('time')['datetime'],
        'views': clean_int(card.select_one('.views').text),
        'url': urljoin(base, link['href']),
        'detail_url': urljoin(base, card.select_one('a.detail')['href']),
    }

record = parse_result_card(first)
print(record)
~~~

> **웹 자동화 안전 한스푼 — 상대 URL**
>
> - **뜻**: /library/web-resource-1처럼 도메인 없이 경로만 있는 링크다.
> - **왜 중요한가**: 그대로 저장하면 어느 사이트의 링크인지 알 수 없다.
> - **수업 기준**: 저장 전 urljoin(base_url, href)로 절대 URL을 만든다.
> - **실수 예시**: 상대 경로만 CSV에 저장해 다음 자동화 단계에서 링크를 열 수 없다.

---

## 5. 한 페이지를 리스트로 정리하기

자동화에서 반복 단위가 정해지면 다음 단계는 리스트를 만드는 것이다. 한 페이지 안의 모든 article.result-card를 parse_result_card 함수로 바꾸면 결과는 딕셔너리 리스트가 된다.

이때 len(records)를 먼저 확인하는 습관이 중요하다. 첫 번째 값만 출력하면 selector가 일부만 맞아도 지나칠 수 있지만, 개수를 확인하면 fixture 구조와 코드가 맞는지 빠르게 판단할 수 있다.

~~~python
page1_records = [parse_result_card(card) for card in cards]
print('records:', len(page1_records))
print(page1_records[0]['title'], page1_records[0]['views'])
print(page1_records[-1]['title'], page1_records[-1]['views'])
~~~

---

## 6. 페이지 번호 규칙을 함수로 분리하기

이번 fixture의 파일명은 search_page_1.html, search_page_2.html, search_page_3.html이다. 페이지 번호가 들어가는 자리를 함수로 분리하면 반복문에서 파일명을 직접 조립하지 않아도 된다.

함수로 분리하는 이유는 단순히 코드가 짧아지기 때문만이 아니다. 실제 사이트에서 페이지 규칙이 바뀌거나 파일명 규칙이 바뀌면 함수 한 곳만 수정하면 된다.

~~~python
def page_filename(page):
    return f'search_page_{page}.html'

for page in range(1, 4):
    print(page, page_filename(page))
~~~

---

## 7. 여러 페이지 순회하기

여러 페이지를 순회할 때는 바깥 반복문이 페이지를 바꾸고, 안쪽 반복문이 카드들을 처리한다. 이 구조를 분명히 이해해야 나중에 페이지네이션이 많은 사이트에서도 무한 반복을 피할 수 있다.

수업에서는 1~3페이지로 고정하지만, 실제 운영 코드에서는 다음 링크 존재 여부, 결과 개수, 최대 페이지 수 같은 중단 기준을 반드시 둔다.

~~~python
all_records = []
for page in range(1, 4):
    page_soup = BeautifulSoup(load_text(page_filename(page)), 'html.parser')
    page_cards = page_soup.select('article.result-card')
    print('page', page, 'cards', len(page_cards))
    for card in page_cards:
        all_records.append(parse_result_card(card))

print('total:', len(all_records))
print(all_records[0]['title'], '->', all_records[-1]['title'])
~~~

> **웹 자동화 안전 한스푼 — 페이지네이션 중단 기준**
>
> - **뜻**: 반복 수집을 언제 멈출지 정하는 규칙이다.
> - **왜 중요한가**: 중단 기준이 없으면 같은 요청을 무한히 반복할 수 있다.
> - **수업 기준**: fixture는 1~3페이지로 제한하고, 실제 사이트는 최대 페이지 수나 다음 링크 존재 여부를 확인한다.
> - **실수 예시**: 무한 반복으로 계속 다음 페이지를 요청하고 빈 결과를 확인하지 않는다.

---

## 8. 다음 페이지 링크 읽기

페이지 번호를 직접 증가시키는 방식과 별개로, HTML 안의 페이지네이션 영역을 읽을 수도 있다. nav.pagination a.next는 다음 페이지 링크를 제공한다. 이 링크에서 다시 query string을 읽으면 다음 page 값을 확인할 수 있다.

~~~python
next_link = soup.select_one('nav.pagination a.next')
next_href = next_link['href']
next_params = parse_qs(urlparse(next_href).query)
print(next_href)
print('next page:', next_params['page'][0])
~~~

---

## 9. 필터링과 집계

수집한 데이터는 저장하기 전에 작은 검증과 요약을 거친다. 예를 들어 조회수 1000 이상인 자료만 골라보거나, 카테고리별 개수를 세면 selector가 의도대로 작동했는지 확인할 수 있다.

~~~python
popular = [row for row in all_records if row['views'] >= 1000]
print('popular:', len(popular))
print([row['title'] for row in popular[:5]])

category_counts = {}
for row in all_records:
    category = row['category']
    category_counts[category] = category_counts.get(category, 0) + 1
print(category_counts)
~~~

---

## 10. 검색 계획 CSV 읽기

search_targets.csv는 검색어, 카테고리, 최소 조회수, 최대 페이지 수를 담은 계획 파일이다. 실제 업무에서는 검색 조건을 코드 안에 박아두기보다 CSV나 설정 파일로 분리하는 편이 관리하기 쉽다.

~~~python
target_rows = list(csv.DictReader(load_text('search_targets.csv').splitlines()))
for row in target_rows:
    print(row['query'], row['category'], row['min_views'], row['max_pages'])
~~~

---

## 11. CSV로 저장하기

마지막으로 여러 페이지에서 모은 데이터를 CSV로 저장한다. 저장 전에는 필드 이름을 먼저 정한다. 필드 이름이 일정해야 나중에 엑셀, 구글시트, 데이터 분석 코드에서 같은 구조로 읽을 수 있다.

~~~python
fieldnames = ['title', 'category', 'page', 'date', 'views', 'url']
with open('lesson02_results.csv', 'w', newline='', encoding='utf-8') as f:
    writer = csv.DictWriter(f, fieldnames=fieldnames)
    writer.writeheader()
    writer.writerows({key: row[key] for key in fieldnames} for row in all_records)
print('saved:', 'lesson02_results.csv', len(all_records))
~~~

---

## 데이터 출처와 안전 규칙

이 레슨의 파일은 모두 수업용 합성 데이터다. 실제 사이트의 개인정보, 로그인 정보, 유료 콘텐츠를 포함하지 않는다. 실제 사이트로 확장할 때는 robots.txt, 이용 약관, 요청 간격, 수집 목적을 먼저 확인한다. 수업 중에는 fixture를 반복 실행하며 구조를 익히고, 외부 사이트를 빠르게 반복 요청하지 않는다.

---

## 12. 디버깅 순서

페이지네이션 자동화가 실패했을 때는 selector부터 고치지 않는다. 먼저 조합된 URL이 의도한 조건을 담고 있는지 확인한다. 그다음 HTML을 읽었는지, 반복 단위 개수가 예상과 맞는지, 카드 안에서 필요한 값이 빠지지 않았는지 차례로 확인한다.

수업 중에는 아래 순서를 그대로 말하게 한다.

1. 현재 page 번호와 파일명 또는 URL을 출력한다.
2. HTML 제목 h1을 출력해 올바른 페이지를 읽었는지 확인한다.
3. article.result-card 개수를 출력한다.
4. 첫 카드의 title, href, data-category를 출력한다.
5. 조회수를 정수로 바꾼 뒤 타입을 확인한다.
6. 저장 전 rows 길이와 첫 행의 key 목록을 확인한다.

이 순서를 지키면 학생이 selector를 무작정 바꾸는 시간을 줄일 수 있다. 특히 페이지 반복에서는 첫 페이지 결과가 세 번 들어가는 실수가 자주 나오므로 page_soup을 반복문 안에서 새로 만드는지 확인한다.

## 13. 실제 사이트로 확장하기 전 점검

이번 레슨은 fixture만 사용하지만, 실제 사이트로 확장할 때는 코드보다 운영 기준을 먼저 정해야 한다. 검색 결과가 공개 페이지인지, robots 정책에서 차단하지 않는지, 요청 간격을 둘 수 있는지, 수집한 값을 저장해도 되는지 확인한다. 학생에게는 “코드가 된다”와 “운영해도 된다”가 다르다는 점을 반복해서 설명한다.

robots_sample.txt는 실제 법적 판단을 대신하지 않는다. 수업에서는 허용/차단/지연 요청의 개념을 익히는 샘플로만 사용한다. 실제 서비스에서는 사이트 약관, 관리자 허가, 개인정보 여부를 함께 검토해야 한다.

## 14. 수업 중 확인 질문

- query string에서 q, category, page는 각각 어떤 역할인가?
- 검색어에 공백이 들어갈 때 urlencode가 필요한 이유는 무엇인가?
- article.result-card 대신 a 태그를 반복 단위로 잡으면 어떤 문제가 생기는가?
- 상대 URL을 그대로 저장하면 다음 자동화 단계에서 어떤 정보가 부족한가?
- range(1, 4)가 1, 2, 3을 만든다는 사실을 어디에서 확인할 수 있는가?
- 조회수 문자열을 정수로 바꾸지 않으면 필터링 결과가 왜 틀릴 수 있는가?
- CSV 저장 전에 rows 길이와 fieldnames를 확인하는 이유는 무엇인가?

## 15. 이번 레슨의 완성 기준

학생이 완성해야 하는 것은 단순히 CSV 파일 하나가 아니다. URL 조건을 구조화하고, 페이지 반복 범위를 통제하고, 카드 데이터를 같은 딕셔너리 구조로 맞춘 뒤, 저장 전 간단한 검증을 하는 흐름이다. 이 네 가지가 연결되어야 다음 레슨의 테이블/리스트 데이터 정리로 자연스럽게 넘어갈 수 있다.

---

## 16. 예제 데이터를 읽는 관찰 포인트

search_page 파일들은 일부러 같은 구조를 유지하면서 값만 다르게 만들었다. 학생은 먼저 “무엇이 반복되고 무엇이 달라지는지”를 말해야 한다. 반복되는 것은 article.result-card, .title a, .views, time 태그이고, 달라지는 것은 data-category, data-page, data-rank, 제목, 조회수다.

이 관찰을 먼저 하지 않으면 selector를 외워서 쓰게 된다. 수업에서는 코드를 치기 전에 HTML 일부를 읽고, 반복 단위와 필요한 필드를 표로 정리하게 한다. 이 과정이 있어야 3강에서 table, list, card 구조가 바뀌어도 같은 방식으로 접근할 수 있다.

## 17. 저장 파일을 검토하는 기준

CSV 저장 후에는 파일이 만들어졌다는 사실만 보지 않는다. 첫 줄에 header가 있는지, title/category/views 같은 필드 이름이 일관적인지, 행 수가 예상한 카드 수와 맞는지 확인한다. 이 레슨에서는 3페이지와 페이지당 9개 카드이므로 전체 rows는 27개가 되어야 한다.

필터링 결과는 기준에 따라 달라질 수 있다. 그래서 최소 조회수 기준을 코드 주석이나 결과 요약에 남겨야 한다. 운영자가 다음에 같은 자동화를 실행할 때 기준을 모르면 결과 차이를 오류로 오해할 수 있다.
