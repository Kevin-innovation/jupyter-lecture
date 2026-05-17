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

### 보강 설명 1

최종 미션은 학생이 자기 노트북에서 실행 결과와 저장 파일을 함께 제출하는 형태로 운영한다. 강사는 결과 파일 이름, 행 수, 요약 문장 세 가지를 우선 확인하면 빠르게 피드백할 수 있다.

### 보강 설명 2

최종 미션은 학생이 자기 노트북에서 실행 결과와 저장 파일을 함께 제출하는 형태로 운영한다. 강사는 결과 파일 이름, 행 수, 요약 문장 세 가지를 우선 확인하면 빠르게 피드백할 수 있다.

### 보강 설명 3

최종 미션은 학생이 자기 노트북에서 실행 결과와 저장 파일을 함께 제출하는 형태로 운영한다. 강사는 결과 파일 이름, 행 수, 요약 문장 세 가지를 우선 확인하면 빠르게 피드백할 수 있다.

### 보강 설명 4

최종 미션은 학생이 자기 노트북에서 실행 결과와 저장 파일을 함께 제출하는 형태로 운영한다. 강사는 결과 파일 이름, 행 수, 요약 문장 세 가지를 우선 확인하면 빠르게 피드백할 수 있다.

### 보강 설명 5

최종 미션은 학생이 자기 노트북에서 실행 결과와 저장 파일을 함께 제출하는 형태로 운영한다. 강사는 결과 파일 이름, 행 수, 요약 문장 세 가지를 우선 확인하면 빠르게 피드백할 수 있다.

### 보강 설명 6

최종 미션은 학생이 자기 노트북에서 실행 결과와 저장 파일을 함께 제출하는 형태로 운영한다. 강사는 결과 파일 이름, 행 수, 요약 문장 세 가지를 우선 확인하면 빠르게 피드백할 수 있다.
