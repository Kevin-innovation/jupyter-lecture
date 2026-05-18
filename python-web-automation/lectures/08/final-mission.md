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

# 레슨 08 — 최종 미션

[![Open in Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/Kevin-innovation/jupyter-lecture/blob/main/python-web-automation/lectures/08/%5B%ED%95%99%EC%83%9D%EC%9A%A9%5D%20%EB%A0%88%EC%8A%A8%2008%20%E2%80%94%20%EC%88%98%EC%A7%91%20%EB%8D%B0%EC%9D%B4%ED%84%B0%20%EA%B2%80%EC%A6%9D%EA%B3%BC%20%EC%A0%80%EC%9E%A5.ipynb)

> 코랩에서 최종 미션을 진행하려면 우측 상단 또는 아래의 **Open in Colab** 버튼을 클릭한다.

## 프로젝트: 수집 데이터 검증과 저장 자동화 리포트

이번 레슨에서 배운 내용을 하나의 작은 운영 자동화로 묶는다. 제공된 fixture만 사용하고 외부 사이트에는 요청하지 않는다.

## 요구사항

1. 레슨의 주요 입력 파일을 모두 읽는다.
2. 최소 2개 이상의 검증 기준을 적용한다.
3. 중복, 오류, 성공 건수를 분리해서 계산한다.
4. 결과 CSV 또는 JSON 중 하나 이상을 저장한다.
5. 마지막에 운영자가 읽을 수 있는 3문장 요약을 작성한다.

## 보너스

- SQLite 저장 또는 상태 코드별 요약을 추가한다.
- 오류가 있는 항목만 따로 저장한다.

## 시작 코드

~~~python
import os
import re
import csv
import json
import time
import sqlite3
import logging
from pathlib import Path
from urllib.parse import urljoin, urlparse

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
    DATA_BASE = 'https://raw.githubusercontent.com/Kevin-innovation/jupyter-lecture/main/python-web-automation/lectures/08/data'
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

def load_csv(filename):
    text = load_text(filename)
    return list(csv.DictReader(text.splitlines()))

def load_json(filename):
    return json.loads(load_text(filename))

def clean_int(text):
    return int(re.sub(r'[^0-9]', '', str(text)) or '0')

def write_csv(path, rows, fieldnames=None):
    rows = list(rows)
    if fieldnames is None:
        fieldnames = list(rows[0].keys()) if rows else []
    with open(path, 'w', encoding='utf-8', newline='') as f:
        writer = csv.DictWriter(f, fieldnames=fieldnames)
        writer.writeheader()
        writer.writerows(rows)
    return path

print('colab:', IS_COLAB)
print('data base:', DATA_BASE)

def parse_price(value):
    return clean_int(value)

def parse_stock(value):
    try:
        return int(str(value).strip())
    except ValueError:
        return None

def validate_feed_row(row, rules):
    errors = []
    for field in rules['required_fields']:
        if not str(row.get(field, '')).strip():
            errors.append(f'missing:{field}')
    price = parse_price(row.get('price_text', ''))
    if price <= 0 or price > rules['max_price']:
        errors.append('invalid:price')
    stock = parse_stock(row.get('stock', ''))
    if stock is None or stock < 0:
        errors.append('invalid:stock')
    if row.get('category') not in rules['valid_categories']:
        errors.append('invalid:category')
    if row.get('status') not in rules['valid_status']:
        errors.append('invalid:status')
    return errors
~~~

## 자동화 결과 요약

- 오늘 확인한 입력:
- 저장한 산출물:
- 다음에 개선할 점:

## 제출 전 자기 점검

최종 미션은 파일을 저장했다는 사실만으로 끝나지 않는다. 제출 전에 다음 항목을 확인한다.

- 원본 행 수와 정제 행 수를 모두 적었다.
- 제외된 행이 있다면 이유를 한 문장으로 설명했다.
- 중복 기준을 무엇으로 잡았는지 적었다.
- 저장 파일 이름을 정확히 적었다.
- 같은 셀을 다시 실행해도 파일이 다시 만들어지는지 확인했다.

운영 요약 3문장은 길게 쓰지 않는다. 첫 문장은 입력 파일과 원본 행 수, 둘째 문장은 정제 결과와 오류 수, 셋째 문장은 다음 실행 때 조심할 점을 적는다. 예시는 다음 형태다.

- raw_product_feed.csv에서 40행을 읽고 필수 컬럼과 가격, 재고, 상태를 검증했다.
- 정제 CSV에는 유효하고 중복되지 않은 행만 저장했고 오류 행은 별도 리포트로 남겼다.
- 다음 실행에서는 category 규칙이 바뀌었는지 먼저 확인한다.
