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

# 레슨 01 — 최종 미션 모범 답안

> 🔒 교사·관리자 전용. 학생에게 배포 금지.

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

## 모범 코드

~~~python
shop_html = load_text('mini_shop.html')
notice_html = load_text('notices.html')
shop_soup = BeautifulSoup(shop_html, 'html.parser')
notice_soup = BeautifulSoup(notice_html, 'html.parser')

products = []
for card in shop_soup.select('.product-card'):
    products.append({
        'name': card.select_one('.name').text.strip(),
        'category': card['data-category'],
        'price': clean_int(card.select_one('.price').text),
        'rating': float(card.select_one('.rating').text.replace('★', '').strip()),
        'stock': clean_int(card.select_one('.stock').text),
        'detail_url': urljoin('https://example.com', card.select_one('a.detail')['href']),
    })

notices = []
for row in notice_soup.select('li.notice-item'):
    notices.append({
        'date': row.select_one('.date').text.strip(),
        'title': row.select_one('.title').text.strip(),
        'department': row.select_one('.department').text.strip(),
        'views': clean_int(row.select_one('.views').text),
    })

with open('lesson01_products.csv', 'w', newline='', encoding='utf-8') as f:
    writer = csv.DictWriter(f, fieldnames=['name', 'category', 'price', 'rating', 'stock', 'detail_url'])
    writer.writeheader()
    writer.writerows(products)

sold_out = [p for p in products if p['stock'] == 0]
top_rated = [p for p in products if p['rating'] >= 4.7]
top_notices = sorted(notices, key=lambda n: n['views'], reverse=True)[:3]

print('상품 수:', len(products))
print('공지 수:', len(notices))
print('품절:', [p['name'] for p in sold_out])
print('고평점:', [p['name'] for p in top_rated])
print('조회수 상위:', [(n['title'], n['views']) for n in top_notices])
~~~

### 왜 이 코드가 정답인지

상품과 공지의 반복 단위가 각각 .product-card와 li.notice-item으로 분리되어 있으므로, 먼저 반복 단위를 선택한 다음 내부의 하위 selector를 읽는 방식이 안정적이다. 가격, 재고, 조회수는 쉼표와 단위가 섞인 문자열이므로 clean_int로 숫자만 남겨야 필터링과 정렬이 올바르게 동작한다. href는 상대 경로이므로 urljoin으로 절대 URL 형태를 만들어 저장한다. 최종 산출물은 CSV 파일과 요약 출력이 함께 있으므로 수집 결과를 확인하고 다른 도구로 이어서 분석하기 쉽다.

### 평가 루브릭

- 필수: 상품/공지 개수 일치, CSV 저장, 품절/고평점/조회수 상위 요약.
- 보너스: 카테고리별 평균 가격, 부서별 공지 개수, 요청 윤리 체크리스트 작성.
- 감점: 숫자 변환 누락, selector 오타, 상대 URL 그대로 저장, 실행 순서 의존으로 재실행 불가.

### 추가 채점 메모

이 최종 미션은 "많이 수집하기"가 아니라 "반복 구조를 정확히 읽고 안전하게 저장하기"를 평가한다. 학생 코드가 모범 답안과 달라도 다음 조건을 만족하면 통과시킨다.

| 항목 | 인정 기준 |
|---|---|
| 상품 추출 | 모든 `.product-card` 를 반복하고 필수 필드 6개를 만든다 |
| 공지 추출 | 모든 `li.notice-item` 을 반복하고 필수 필드 4개를 만든다 |
| 숫자 변환 | 가격, 재고, 조회수는 문자열이 아니라 숫자로 비교 가능하다 |
| URL 처리 | 상세 링크는 `urljoin` 또는 동등한 방식으로 절대 URL이 된다 |
| 저장 | CSV가 열렸을 때 헤더와 행 수가 확인된다 |
| 안전 메모 | 실제 사이트 적용 전 확인할 운영 규칙이 들어 있다 |

학생이 `pandas` 로 CSV를 저장해도 결과 파일이 같으면 인정할 수 있다. 다만 1강에서는 표준 라이브러리 `csv` 로도 충분히 가능하다는 점을 한 번 더 설명한다. 학생이 실제 사이트 URL을 임의로 넣어 반복 요청했다면 결과가 맞아도 안전 기준 미달로 보완시킨다.

### 수업 후 확장 질문

빠른 학생에게는 다음 질문을 던진다.

1. 상품 카드가 1,000개라면 print를 어디까지 줄여야 할까?
2. `.price` class 이름이 `.product-price` 로 바뀌면 어떤 부분이 깨질까?
3. CSV를 매일 저장한다면 파일명에 날짜를 넣어야 하는 이유는 무엇일까?
4. 외부 사이트에서 429 상태 코드가 나오면 코드를 어떻게 멈추거나 늦출 수 있을까?
