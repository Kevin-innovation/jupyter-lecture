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

# 레슨 02 — 최종 미션 모범 답안

> 교사·관리자 전용. 학생에게 배포 금지.

이 답안은 URL 파라미터, 페이지 반복, 상대 URL 변환, 필터링, CSV 저장을 한 흐름으로 묶는다.

## 모범 답안

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

target_rows = list(csv.DictReader(load_text('search_targets.csv').splitlines()))
all_rows = []
for page in range(1, 4):
    page_soup = BeautifulSoup(load_text(page_filename(page)), 'html.parser')
    for card in page_soup.select('article.result-card'):
        all_rows.append(parse_result_card(card))

min_views_by_category = {row['category']: int(row['min_views']) for row in target_rows}
filtered_rows = [row for row in all_rows if row['views'] >= min_views_by_category.get(row['category'], 0)]

fieldnames = ['title', 'category', 'page', 'rank', 'date', 'views', 'url', 'detail_url']
with open('lesson02_all_results.csv', 'w', newline='', encoding='utf-8') as f:
    writer = csv.DictWriter(f, fieldnames=fieldnames)
    writer.writeheader()
    writer.writerows(all_rows)

with open('lesson02_filtered_results.csv', 'w', newline='', encoding='utf-8') as f:
    writer = csv.DictWriter(f, fieldnames=fieldnames)
    writer.writeheader()
    writer.writerows(filtered_rows)

category_summary = {}
for row in all_rows:
    item = category_summary.setdefault(row['category'], {'count': 0, 'views': 0})
    item['count'] += 1
    item['views'] += row['views']

for category, item in category_summary.items():
    print(category, item['count'], round(item['views'] / item['count'], 1))
print('all:', len(all_rows))
print('filtered:', len(filtered_rows))
~~~

## 왜 이 답안이 기준을 충족하는가

- 검색 조건과 페이지 파일명을 코드에서 분리해 다음 실행 때 수정 지점이 분명하다.
- 카드 하나를 딕셔너리로 바꾸는 함수를 만들어 전체 페이지 반복에서도 같은 구조를 유지한다.
- 상대 링크를 절대 URL로 변환해 CSV만 열어도 이동 가능한 링크가 남는다.
- 저장 전 필터링 기준을 search_targets.csv에서 읽어 운영자가 조건을 파일로 조정할 수 있다.

## 채점 메모

- 학생 답안이 결과를 맞히더라도 URL 변환, 조회수 정수 변환, CSV 헤더가 빠졌으면 감점한다.
- 실제 사이트 URL을 직접 반복 요청한 답안은 이 레슨 기준에서 실패로 본다.
- 필터링 기준을 코드에 하드코딩했더라도 기본 동작은 인정하되, 검색 계획 CSV를 읽지 않았다면 운영 자동화 관점에서 감점한다.

---

## 운영 해설

이 최종 답안은 검색 계획 파일과 페이지 fixture를 분리해 둔다. 운영자가 검색 조건을 바꾸고 싶을 때는 search_targets.csv만 수정하면 되고, 카드 파싱 로직은 유지된다. 실제 사이트로 확장할 때도 이 구조를 유지하면 요청 대상, 반복 범위, 저장 결과를 따로 검토할 수 있다.

학생 답안을 볼 때는 filtered_rows의 개수보다 필터링 기준이 어디에서 왔는지를 먼저 확인한다. 기준을 코드 안에 숫자로 고정해도 예제는 맞을 수 있지만, 운영 자동화에서는 조건을 파일이나 설정으로 분리하는 편이 재사용성이 높다.
