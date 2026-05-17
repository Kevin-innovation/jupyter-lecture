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

# 레슨 08 — 수집 데이터 검증과 저장

[![Open in Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/Kevin-innovation/jupyter-lecture/blob/main/python-web-automation/lectures/08/[학생용] 레슨 08 — 수집 데이터 검증과 저장.ipynb)

이 노트북은 읽기와 따라하기용 강의 노트북이다. 수집한 데이터를 바로 저장하지 않고 스키마, 누락, 중복, 범위 오류를 점검한 뒤 CSV, JSON, SQLite로 남기는 파이프라인을 안전한 합성 fixture로 연습한다.

## 학습 목표

1. 수집 행의 필수 컬럼과 값 범위를 검증한다.
2. 누락, 중복, 형식 오류를 오류 코드로 분류한다.
3. 정제된 데이터와 품질 리포트를 분리 저장한다.
4. CSV, JSON, SQLite 저장 방식의 차이를 설명한다.
5. 저장 전에 검증 로그를 남기는 운영 습관을 만든다.

---

## 1. 수업 맥락과 안전 기준

학원 운영자가 여러 공지/상품/자료실 페이지에서 가져온 데이터를 그대로 쓰면 중복 공지, 가격 형식 오류, 비공개 상태 노출이 섞일 수 있다. 이 레슨은 “수집했다”보다 “믿고 저장할 수 있다”를 기준으로 자동화를 마무리하는 훈련이다.

자동화는 빠르게 반복하는 도구이기 때문에 실패했을 때 더 위험해질 수 있다. 그래서 이번 레슨에서는 모든 입력을 수업용 파일로 고정하고, 결과를 저장하기 전에 검증하거나 로그를 남기는 과정을 코드에 포함한다. 이 습관은 실제 사이트를 대상으로 할 때 요청량을 줄이고, 오류를 빨리 발견하게 만든다.

## 2. 환경 셀

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

---

## 3. 핵심 개념

아래 셀은 강사가 먼저 실행 흐름을 보여주고, 학생이 같은 구조를 자기 말로 설명하도록 만든 예제다. 코드는 짧지만 입력, 처리, 출력이 분리되어 있어 나중에 문제 풀이로 확장하기 쉽다.

~~~python
rows = load_csv('raw_product_feed.csv')
rules = load_json('category_rules.json')
print(len(rows), rules['required_fields'])
~~~
이 셀에서 확인해야 할 것은 출력값 하나가 아니라 데이터가 어떤 단계에서 바뀌었는지다. 자동화 수업에서는 중간 변수명을 명확히 두면 학생이 오류 위치를 쉽게 찾을 수 있다.

---

## 4. 자료 구조 확인

아래 셀은 강사가 먼저 실행 흐름을 보여주고, 학생이 같은 구조를 자기 말로 설명하도록 만든 예제다. 코드는 짧지만 입력, 처리, 출력이 분리되어 있어 나중에 문제 풀이로 확장하기 쉽다.

~~~python
sample = rows[0]
print(sample)
print(parse_price(sample['price_text']), parse_stock(sample['stock']))
~~~
이 셀에서 확인해야 할 것은 출력값 하나가 아니라 데이터가 어떤 단계에서 바뀌었는지다. 자동화 수업에서는 중간 변수명을 명확히 두면 학생이 오류 위치를 쉽게 찾을 수 있다.

---

## 5. 품질 기준 적용

아래 셀은 강사가 먼저 실행 흐름을 보여주고, 학생이 같은 구조를 자기 말로 설명하도록 만든 예제다. 코드는 짧지만 입력, 처리, 출력이 분리되어 있어 나중에 문제 풀이로 확장하기 쉽다.

~~~python
checked = []
for row in rows:
    errors = validate_feed_row(row, rules)
    checked.append({**row, 'errors': '|'.join(errors), 'is_valid': not errors})
print(checked[0])
~~~
이 셀에서 확인해야 할 것은 출력값 하나가 아니라 데이터가 어떤 단계에서 바뀌었는지다. 자동화 수업에서는 중간 변수명을 명확히 두면 학생이 오류 위치를 쉽게 찾을 수 있다.

---

## 6. 저장과 보고

아래 셀은 강사가 먼저 실행 흐름을 보여주고, 학생이 같은 구조를 자기 말로 설명하도록 만든 예제다. 코드는 짧지만 입력, 처리, 출력이 분리되어 있어 나중에 문제 풀이로 확장하기 쉽다.

~~~python
seen = set()
unique_rows = []
duplicates = []
for row in checked:
    key = row['record_id']
    if key in seen:
        duplicates.append(key)
    else:
        seen.add(key)
        unique_rows.append(row)
print('unique:', len(unique_rows), 'duplicates:', duplicates[:3])
~~~
이 셀에서 확인해야 할 것은 출력값 하나가 아니라 데이터가 어떤 단계에서 바뀌었는지다. 자동화 수업에서는 중간 변수명을 명확히 두면 학생이 오류 위치를 쉽게 찾을 수 있다.

---

## 7. 운영 관점 점검

아래 셀은 강사가 먼저 실행 흐름을 보여주고, 학생이 같은 구조를 자기 말로 설명하도록 만든 예제다. 코드는 짧지만 입력, 처리, 출력이 분리되어 있어 나중에 문제 풀이로 확장하기 쉽다.

~~~python
clean_rows = []
for row in unique_rows:
    if row['is_valid']:
        clean_rows.append({
            'record_id': row['record_id'],
            'title': row['title'].strip(),
            'category': row['category'],
            'price': parse_price(row['price_text']),
            'stock': parse_stock(row['stock']),
            'status': row['status'],
        })
print(clean_rows[:2])
~~~
이 셀에서 확인해야 할 것은 출력값 하나가 아니라 데이터가 어떤 단계에서 바뀌었는지다. 자동화 수업에서는 중간 변수명을 명확히 두면 학생이 오류 위치를 쉽게 찾을 수 있다.

---

## 8. 마무리 체크

아래 셀은 강사가 먼저 실행 흐름을 보여주고, 학생이 같은 구조를 자기 말로 설명하도록 만든 예제다. 코드는 짧지만 입력, 처리, 출력이 분리되어 있어 나중에 문제 풀이로 확장하기 쉽다.

~~~python
write_csv('lesson08_clean_feed.csv', clean_rows)
report = {'raw_count': len(rows), 'clean_count': len(clean_rows), 'duplicate_count': len(duplicates)}
Path('lesson08_quality_report.json').write_text(json.dumps(report, ensure_ascii=False, indent=2), encoding='utf-8')
print(report)
~~~
이 셀에서 확인해야 할 것은 출력값 하나가 아니라 데이터가 어떤 단계에서 바뀌었는지다. 자동화 수업에서는 중간 변수명을 명확히 두면 학생이 오류 위치를 쉽게 찾을 수 있다.

---

## 9. 핵심 개념

아래 셀은 강사가 먼저 실행 흐름을 보여주고, 학생이 같은 구조를 자기 말로 설명하도록 만든 예제다. 코드는 짧지만 입력, 처리, 출력이 분리되어 있어 나중에 문제 풀이로 확장하기 쉽다.

~~~python
conn = sqlite3.connect('lesson08_feed.db')
conn.execute('drop table if exists feed')
conn.execute('create table feed(record_id text, title text, category text, price integer, stock integer, status text)')
conn.executemany('insert into feed values(:record_id, :title, :category, :price, :stock, :status)', clean_rows)
print(conn.execute('select category, count(*) from feed group by category').fetchall())
conn.close()
~~~
이 셀에서 확인해야 할 것은 출력값 하나가 아니라 데이터가 어떤 단계에서 바뀌었는지다. 자동화 수업에서는 중간 변수명을 명확히 두면 학생이 오류 위치를 쉽게 찾을 수 있다.

---

## 데이터 출처와 안전 규칙

raw_product_feed.csv는 의도적으로 누락, 중복, 형식 오류가 섞인 합성 상품 피드다. category_rules.json은 허용 카테고리, 상태, 최대 가격 기준을 담는다. validation_panel.html은 같은 데이터를 웹 표 형태로 확인하는 연습용 페이지다. schema_notes.txt는 사람이 읽는 스키마 설명이다.

- 모든 파일은 수업용 합성 데이터다.
- 실제 사이트에 반복 요청하지 않는다.
- 저장 파일은 레슨 폴더 또는 코랩 현재 작업 폴더에만 만든다.
- 외부 사이트로 확장할 때는 약관, robots.txt, 요청 간격, 개인정보 포함 여부를 먼저 확인한다.

### 보강 설명 1

학생에게는 코드의 속도보다 검증 순서를 강조한다. 입력을 읽고, 구조를 확인하고, 오류를 분류하고, 저장한 뒤 요약을 남기는 순서가 유지되면 도구가 바뀌어도 자동화 품질은 크게 흔들리지 않는다.

### 보강 설명 2

학생에게는 코드의 속도보다 검증 순서를 강조한다. 입력을 읽고, 구조를 확인하고, 오류를 분류하고, 저장한 뒤 요약을 남기는 순서가 유지되면 도구가 바뀌어도 자동화 품질은 크게 흔들리지 않는다.

### 보강 설명 3

학생에게는 코드의 속도보다 검증 순서를 강조한다. 입력을 읽고, 구조를 확인하고, 오류를 분류하고, 저장한 뒤 요약을 남기는 순서가 유지되면 도구가 바뀌어도 자동화 품질은 크게 흔들리지 않는다.

### 보강 설명 4

학생에게는 코드의 속도보다 검증 순서를 강조한다. 입력을 읽고, 구조를 확인하고, 오류를 분류하고, 저장한 뒤 요약을 남기는 순서가 유지되면 도구가 바뀌어도 자동화 품질은 크게 흔들리지 않는다.

### 보강 설명 5

학생에게는 코드의 속도보다 검증 순서를 강조한다. 입력을 읽고, 구조를 확인하고, 오류를 분류하고, 저장한 뒤 요약을 남기는 순서가 유지되면 도구가 바뀌어도 자동화 품질은 크게 흔들리지 않는다.

### 보강 설명 6

학생에게는 코드의 속도보다 검증 순서를 강조한다. 입력을 읽고, 구조를 확인하고, 오류를 분류하고, 저장한 뒤 요약을 남기는 순서가 유지되면 도구가 바뀌어도 자동화 품질은 크게 흔들리지 않는다.

### 보강 설명 7

학생에게는 코드의 속도보다 검증 순서를 강조한다. 입력을 읽고, 구조를 확인하고, 오류를 분류하고, 저장한 뒤 요약을 남기는 순서가 유지되면 도구가 바뀌어도 자동화 품질은 크게 흔들리지 않는다.

### 보강 설명 8

학생에게는 코드의 속도보다 검증 순서를 강조한다. 입력을 읽고, 구조를 확인하고, 오류를 분류하고, 저장한 뒤 요약을 남기는 순서가 유지되면 도구가 바뀌어도 자동화 품질은 크게 흔들리지 않는다.

### 보강 설명 9

학생에게는 코드의 속도보다 검증 순서를 강조한다. 입력을 읽고, 구조를 확인하고, 오류를 분류하고, 저장한 뒤 요약을 남기는 순서가 유지되면 도구가 바뀌어도 자동화 품질은 크게 흔들리지 않는다.

### 보강 설명 10

학생에게는 코드의 속도보다 검증 순서를 강조한다. 입력을 읽고, 구조를 확인하고, 오류를 분류하고, 저장한 뒤 요약을 남기는 순서가 유지되면 도구가 바뀌어도 자동화 품질은 크게 흔들리지 않는다.

### 보강 설명 11

학생에게는 코드의 속도보다 검증 순서를 강조한다. 입력을 읽고, 구조를 확인하고, 오류를 분류하고, 저장한 뒤 요약을 남기는 순서가 유지되면 도구가 바뀌어도 자동화 품질은 크게 흔들리지 않는다.

### 보강 설명 12

학생에게는 코드의 속도보다 검증 순서를 강조한다. 입력을 읽고, 구조를 확인하고, 오류를 분류하고, 저장한 뒤 요약을 남기는 순서가 유지되면 도구가 바뀌어도 자동화 품질은 크게 흔들리지 않는다.

### 보강 설명 13

학생에게는 코드의 속도보다 검증 순서를 강조한다. 입력을 읽고, 구조를 확인하고, 오류를 분류하고, 저장한 뒤 요약을 남기는 순서가 유지되면 도구가 바뀌어도 자동화 품질은 크게 흔들리지 않는다.

### 보강 설명 14

학생에게는 코드의 속도보다 검증 순서를 강조한다. 입력을 읽고, 구조를 확인하고, 오류를 분류하고, 저장한 뒤 요약을 남기는 순서가 유지되면 도구가 바뀌어도 자동화 품질은 크게 흔들리지 않는다.

### 보강 설명 15

학생에게는 코드의 속도보다 검증 순서를 강조한다. 입력을 읽고, 구조를 확인하고, 오류를 분류하고, 저장한 뒤 요약을 남기는 순서가 유지되면 도구가 바뀌어도 자동화 품질은 크게 흔들리지 않는다.

### 보강 설명 16

학생에게는 코드의 속도보다 검증 순서를 강조한다. 입력을 읽고, 구조를 확인하고, 오류를 분류하고, 저장한 뒤 요약을 남기는 순서가 유지되면 도구가 바뀌어도 자동화 품질은 크게 흔들리지 않는다.

### 보강 설명 17

학생에게는 코드의 속도보다 검증 순서를 강조한다. 입력을 읽고, 구조를 확인하고, 오류를 분류하고, 저장한 뒤 요약을 남기는 순서가 유지되면 도구가 바뀌어도 자동화 품질은 크게 흔들리지 않는다.

### 보강 설명 18

학생에게는 코드의 속도보다 검증 순서를 강조한다. 입력을 읽고, 구조를 확인하고, 오류를 분류하고, 저장한 뒤 요약을 남기는 순서가 유지되면 도구가 바뀌어도 자동화 품질은 크게 흔들리지 않는다.

### 보강 설명 19

학생에게는 코드의 속도보다 검증 순서를 강조한다. 입력을 읽고, 구조를 확인하고, 오류를 분류하고, 저장한 뒤 요약을 남기는 순서가 유지되면 도구가 바뀌어도 자동화 품질은 크게 흔들리지 않는다.

### 보강 설명 20

학생에게는 코드의 속도보다 검증 순서를 강조한다. 입력을 읽고, 구조를 확인하고, 오류를 분류하고, 저장한 뒤 요약을 남기는 순서가 유지되면 도구가 바뀌어도 자동화 품질은 크게 흔들리지 않는다.

### 보강 설명 21

학생에게는 코드의 속도보다 검증 순서를 강조한다. 입력을 읽고, 구조를 확인하고, 오류를 분류하고, 저장한 뒤 요약을 남기는 순서가 유지되면 도구가 바뀌어도 자동화 품질은 크게 흔들리지 않는다.

### 보강 설명 22

학생에게는 코드의 속도보다 검증 순서를 강조한다. 입력을 읽고, 구조를 확인하고, 오류를 분류하고, 저장한 뒤 요약을 남기는 순서가 유지되면 도구가 바뀌어도 자동화 품질은 크게 흔들리지 않는다.

### 보강 설명 23

학생에게는 코드의 속도보다 검증 순서를 강조한다. 입력을 읽고, 구조를 확인하고, 오류를 분류하고, 저장한 뒤 요약을 남기는 순서가 유지되면 도구가 바뀌어도 자동화 품질은 크게 흔들리지 않는다.

### 보강 설명 24

학생에게는 코드의 속도보다 검증 순서를 강조한다. 입력을 읽고, 구조를 확인하고, 오류를 분류하고, 저장한 뒤 요약을 남기는 순서가 유지되면 도구가 바뀌어도 자동화 품질은 크게 흔들리지 않는다.

### 보강 설명 25

학생에게는 코드의 속도보다 검증 순서를 강조한다. 입력을 읽고, 구조를 확인하고, 오류를 분류하고, 저장한 뒤 요약을 남기는 순서가 유지되면 도구가 바뀌어도 자동화 품질은 크게 흔들리지 않는다.

### 보강 설명 26

학생에게는 코드의 속도보다 검증 순서를 강조한다. 입력을 읽고, 구조를 확인하고, 오류를 분류하고, 저장한 뒤 요약을 남기는 순서가 유지되면 도구가 바뀌어도 자동화 품질은 크게 흔들리지 않는다.
