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

# 레슨 01 — 최종 미션: 샘플 쇼핑/공지 페이지 수집 리포트

이번 최종 미션은 mini_shop.html과 notices.html을 하나의 작은 수집 리포트로 정리하는 것이다. 실제 사이트가 아니라 수업용 합성 HTML이므로 요청 제한 문제 없이 반복 연습할 수 있다.

## 목표

1. 상품 카드 전체를 파싱해 products 리스트를 만든다.
2. 상품 결과를 lesson01_products.csv로 저장한다.
3. 공지 목록 전체를 파싱해 notices 리스트를 만든다.
4. 품절 상품, 고평점 상품, 조회수 상위 공지를 요약한다.
5. 마지막 마크다운 셀에 자동화 윤리 체크리스트를 작성한다.

## 환경 셀

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

## 진행 셀

~~~python
# 1. HTML 읽기
shop_html = load_text('mini_shop.html')
notice_html = load_text('notices.html')

# 2. BeautifulSoup 객체 만들기
shop_soup = BeautifulSoup(____, 'html.parser')
notice_soup = BeautifulSoup(____, 'html.parser')

# 3. 상품 추출
products = []
for card in shop_soup.select('____'):
    products.append({
        'name': ____,
        'category': ____,
        'price': ____,
        'rating': ____,
        'stock': ____,
        'detail_url': ____,
    })

# 4. 공지 추출
notices = []
for row in notice_soup.select('____'):
    notices.append({
        'date': ____,
        'title': ____,
        'department': ____,
        'views': ____,
    })

print(len(products), len(notices))
~~~

## CSV 저장 셀

~~~python
# products를 CSV로 저장하세요.
with open('lesson01_products.csv', 'w', newline='', encoding='utf-8') as f:
    writer = csv.DictWriter(f, fieldnames=['name', 'category', 'price', 'rating', 'stock', 'detail_url'])
    writer.writeheader()
    writer.writerows(____)
print('saved')
~~~

## 요약 셀

~~~python
# 품절 상품, 고평점 상품, 조회수 상위 공지를 요약하세요.
sold_out = ____
top_rated = ____
top_notices = ____

print('품절 상품:', ____)
print('고평점 상품:', ____)
print('조회수 상위 공지:', ____)
~~~

## 결론 작성

아래 항목을 짧게 작성한다.

- 수집한 상품 수 / 공지 수
- 운영자가 먼저 봐야 할 품절 또는 고평점 상품
- 조회수 상위 공지가 의미하는 것
- 실제 사이트에서 같은 코드를 실행하기 전에 확인해야 할 윤리/안전 항목 3개

## 필수 요구사항

최종 미션은 아래 5개를 모두 만족해야 완료로 본다.

1. `mini_shop.html` 의 모든 상품 카드를 빠짐없이 읽어 `products` 리스트를 만든다.
2. `notices.html` 의 모든 공지 행을 빠짐없이 읽어 `notices` 리스트를 만든다.
3. 상품 결과를 `lesson01_products.csv` 로 저장하고, 헤더가 포함되어야 한다.
4. 품절 상품, 평점 4.7 이상 상품, 조회수 상위 공지 3개를 각각 출력한다.
5. 마지막 마크다운 셀에 실제 사이트 적용 전 안전 체크 3가지를 적는다.

## 보너스 요구사항

시간이 남는 학생은 아래 중 1개 이상을 추가한다.

- 카테고리별 상품 수와 평균 가격을 계산한다.
- 부서별 공지 수와 평균 조회수를 계산한다.
- `targets.csv` 에서 allowed가 `yes` 인 항목만 읽어 처리 대상 목록을 만든다.
- 결과 CSV 파일명을 날짜가 포함된 형태로 바꾼다. 예: `lesson01_products_2026-05-17.csv`

## 제출 전 검증

제출 직전에는 런타임을 다시 시작한 뒤 위에서부터 다시 실행한다. 중간 셀부터 실행해야만 동작하는 노트북은 완료로 보지 않는다.

~~~python
# 선택 검증 셀
print('상품 수:', len(products))
print('공지 수:', len(notices))
print('CSV 저장 대상 행 수:', len(products))
print('품절 상품 수:', len(sold_out))
print('고평점 상품 수:', len(top_rated))
print('조회수 상위 공지 수:', len(top_notices))
~~~

## 평가 루브릭

| 항목 | 기준 | 점수 |
|---|---|---:|
| 데이터 로드 | HTML 두 개를 모두 읽고 BeautifulSoup 객체로 변환 | 2 |
| 상품 파싱 | name/category/price/rating/stock/detail_url 추출 | 2 |
| 공지 파싱 | date/title/department/views 추출 | 2 |
| 저장 | CSV 헤더와 행이 정상 저장 | 1 |
| 요약 | 품절/고평점/조회수 상위 결과가 문장으로 정리 | 2 |
| 안전 메모 | robots.txt, 요청 간격, 개인정보/약관 확인 언급 | 1 |

8점 이상이면 완료, 10점이면 우수로 본다.
