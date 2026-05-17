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

# 레슨 08 — 실습 문제 정답지

> 🔒 교사·관리자 전용. 학생에게 배포 금지.

수집 데이터 검증과 저장 실습 문제의 모범 답안이다. 출력값만 보지 말고 검증 기준, 저장 구조, 로그를 함께 확인한다.

## 0. 환경 셀

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

## 문제 1 정답 — 원본 피드 행 수 확인

~~~python
rows = load_csv('raw_product_feed.csv')
print(len(rows))
~~~

### 왜 이 코드가 정답인지

이 답안은 문제에서 요구한 입력 구조를 먼저 읽고, 필요한 컬럼이나 selector만 선택한 뒤 결과를 명시적인 변수에 저장한다. `원본 피드 행 수 확인` 단계에서 중요한 점은 출력 하나를 맞히는 것이 아니라 이후 문제에서 재사용할 수 있는 안정적인 중간 결과를 남기는 것이다. 빈칸에는 파일명, 컬럼명, 키 이름, 반복 제어 중 핵심 위치만 남겼기 때문에 학생이 데이터 구조를 이해해야 완성할 수 있다. 자주 보이는 오답은 컬럼명을 추측하거나, 중복과 실패 상태를 처리하지 않고 바로 저장하는 방식이다.

---

## 문제 2 정답 — 검증 규칙 JSON 읽기

~~~python
rules = load_json('category_rules.json')
print(rules['required_fields'])
~~~

### 왜 이 코드가 정답인지

이 답안은 문제에서 요구한 입력 구조를 먼저 읽고, 필요한 컬럼이나 selector만 선택한 뒤 결과를 명시적인 변수에 저장한다. `검증 규칙 JSON 읽기` 단계에서 중요한 점은 출력 하나를 맞히는 것이 아니라 이후 문제에서 재사용할 수 있는 안정적인 중간 결과를 남기는 것이다. 빈칸에는 파일명, 컬럼명, 키 이름, 반복 제어 중 핵심 위치만 남겼기 때문에 학생이 데이터 구조를 이해해야 완성할 수 있다. 자주 보이는 오답은 컬럼명을 추측하거나, 중복과 실패 상태를 처리하지 않고 바로 저장하는 방식이다.

---

## 문제 3 정답 — 가격 문자열을 숫자로 변환

~~~python
price = parse_price(rows[0]['price_text'])
print(price)
~~~

### 왜 이 코드가 정답인지

이 답안은 문제에서 요구한 입력 구조를 먼저 읽고, 필요한 컬럼이나 selector만 선택한 뒤 결과를 명시적인 변수에 저장한다. `가격 문자열을 숫자로 변환` 단계에서 중요한 점은 출력 하나를 맞히는 것이 아니라 이후 문제에서 재사용할 수 있는 안정적인 중간 결과를 남기는 것이다. 빈칸에는 파일명, 컬럼명, 키 이름, 반복 제어 중 핵심 위치만 남겼기 때문에 학생이 데이터 구조를 이해해야 완성할 수 있다. 자주 보이는 오답은 컬럼명을 추측하거나, 중복과 실패 상태를 처리하지 않고 바로 저장하는 방식이다.

---

## 문제 4 정답 — 재고 값을 정수로 변환

~~~python
stock = parse_stock(rows[0]['stock'])
print(stock)
~~~

### 왜 이 코드가 정답인지

이 답안은 문제에서 요구한 입력 구조를 먼저 읽고, 필요한 컬럼이나 selector만 선택한 뒤 결과를 명시적인 변수에 저장한다. `재고 값을 정수로 변환` 단계에서 중요한 점은 출력 하나를 맞히는 것이 아니라 이후 문제에서 재사용할 수 있는 안정적인 중간 결과를 남기는 것이다. 빈칸에는 파일명, 컬럼명, 키 이름, 반복 제어 중 핵심 위치만 남겼기 때문에 학생이 데이터 구조를 이해해야 완성할 수 있다. 자주 보이는 오답은 컬럼명을 추측하거나, 중복과 실패 상태를 처리하지 않고 바로 저장하는 방식이다.

---

## 문제 5 정답 — 첫 행 오류 코드 확인

~~~python
errors = validate_feed_row(rows[0], rules)
print(errors)
~~~

### 왜 이 코드가 정답인지

이 답안은 문제에서 요구한 입력 구조를 먼저 읽고, 필요한 컬럼이나 selector만 선택한 뒤 결과를 명시적인 변수에 저장한다. `첫 행 오류 코드 확인` 단계에서 중요한 점은 출력 하나를 맞히는 것이 아니라 이후 문제에서 재사용할 수 있는 안정적인 중간 결과를 남기는 것이다. 빈칸에는 파일명, 컬럼명, 키 이름, 반복 제어 중 핵심 위치만 남겼기 때문에 학생이 데이터 구조를 이해해야 완성할 수 있다. 자주 보이는 오답은 컬럼명을 추측하거나, 중복과 실패 상태를 처리하지 않고 바로 저장하는 방식이다.

---

## 문제 6 정답 — 전체 행 검증 결과 만들기

~~~python
checked = []
for row in rows:
    errors = validate_feed_row(row, rules)
    checked.append({**row, 'errors': '|'.join(errors), 'is_valid': not errors})
print(len(checked))
~~~

### 왜 이 코드가 정답인지

이 답안은 문제에서 요구한 입력 구조를 먼저 읽고, 필요한 컬럼이나 selector만 선택한 뒤 결과를 명시적인 변수에 저장한다. `전체 행 검증 결과 만들기` 단계에서 중요한 점은 출력 하나를 맞히는 것이 아니라 이후 문제에서 재사용할 수 있는 안정적인 중간 결과를 남기는 것이다. 빈칸에는 파일명, 컬럼명, 키 이름, 반복 제어 중 핵심 위치만 남겼기 때문에 학생이 데이터 구조를 이해해야 완성할 수 있다. 자주 보이는 오답은 컬럼명을 추측하거나, 중복과 실패 상태를 처리하지 않고 바로 저장하는 방식이다.

---

## 문제 7 정답 — 중복 record_id 찾기

~~~python
seen = set()
duplicates = []
for row in checked:
    key = row['record_id']
    if key in seen:
        duplicates.append(key)
    seen.add(key)
print(duplicates[:5])
~~~

### 왜 이 코드가 정답인지

이 답안은 문제에서 요구한 입력 구조를 먼저 읽고, 필요한 컬럼이나 selector만 선택한 뒤 결과를 명시적인 변수에 저장한다. `중복 record_id 찾기` 단계에서 중요한 점은 출력 하나를 맞히는 것이 아니라 이후 문제에서 재사용할 수 있는 안정적인 중간 결과를 남기는 것이다. 빈칸에는 파일명, 컬럼명, 키 이름, 반복 제어 중 핵심 위치만 남겼기 때문에 학생이 데이터 구조를 이해해야 완성할 수 있다. 자주 보이는 오답은 컬럼명을 추측하거나, 중복과 실패 상태를 처리하지 않고 바로 저장하는 방식이다.

---

## 문제 8 정답 — 유효 행만 정제 리스트로 만들기

~~~python
clean_rows = []
seen = set()
for row in checked:
    if row['record_id'] in seen:
        continue
    seen.add(row['record_id'])
    if row['is_valid']:
        clean_rows.append({'record_id': row['record_id'], 'title': row['title'].strip(), 'category': row['category'], 'price': parse_price(row['price_text']), 'stock': parse_stock(row['stock']), 'status': row['status']})
print(len(clean_rows))
~~~

### 왜 이 코드가 정답인지

이 답안은 문제에서 요구한 입력 구조를 먼저 읽고, 필요한 컬럼이나 selector만 선택한 뒤 결과를 명시적인 변수에 저장한다. `유효 행만 정제 리스트로 만들기` 단계에서 중요한 점은 출력 하나를 맞히는 것이 아니라 이후 문제에서 재사용할 수 있는 안정적인 중간 결과를 남기는 것이다. 빈칸에는 파일명, 컬럼명, 키 이름, 반복 제어 중 핵심 위치만 남겼기 때문에 학생이 데이터 구조를 이해해야 완성할 수 있다. 자주 보이는 오답은 컬럼명을 추측하거나, 중복과 실패 상태를 처리하지 않고 바로 저장하는 방식이다.

---

## 문제 9 정답 — 카테고리별 개수 요약

~~~python
summary = {}
for row in clean_rows:
    summary[row['category']] = summary.get(row['category'], 0) + 1
print(summary)
~~~

### 왜 이 코드가 정답인지

이 답안은 문제에서 요구한 입력 구조를 먼저 읽고, 필요한 컬럼이나 selector만 선택한 뒤 결과를 명시적인 변수에 저장한다. `카테고리별 개수 요약` 단계에서 중요한 점은 출력 하나를 맞히는 것이 아니라 이후 문제에서 재사용할 수 있는 안정적인 중간 결과를 남기는 것이다. 빈칸에는 파일명, 컬럼명, 키 이름, 반복 제어 중 핵심 위치만 남겼기 때문에 학생이 데이터 구조를 이해해야 완성할 수 있다. 자주 보이는 오답은 컬럼명을 추측하거나, 중복과 실패 상태를 처리하지 않고 바로 저장하는 방식이다.

---

## 문제 10 정답 — 정제 CSV 저장

~~~python
write_csv('lesson08_clean_feed.csv', clean_rows)
print(Path('lesson08_clean_feed.csv').exists())
~~~

### 왜 이 코드가 정답인지

이 답안은 문제에서 요구한 입력 구조를 먼저 읽고, 필요한 컬럼이나 selector만 선택한 뒤 결과를 명시적인 변수에 저장한다. `정제 CSV 저장` 단계에서 중요한 점은 출력 하나를 맞히는 것이 아니라 이후 문제에서 재사용할 수 있는 안정적인 중간 결과를 남기는 것이다. 빈칸에는 파일명, 컬럼명, 키 이름, 반복 제어 중 핵심 위치만 남겼기 때문에 학생이 데이터 구조를 이해해야 완성할 수 있다. 자주 보이는 오답은 컬럼명을 추측하거나, 중복과 실패 상태를 처리하지 않고 바로 저장하는 방식이다.

---

## 문제 11 정답 — 품질 리포트 JSON 저장

~~~python
report = {'raw_count': len(rows), 'clean_count': len(clean_rows), 'duplicate_count': len(duplicates)}
Path('lesson08_quality_report.json').write_text(json.dumps(report, ensure_ascii=False, indent=2), encoding='utf-8')
print(report)
~~~

### 왜 이 코드가 정답인지

이 답안은 문제에서 요구한 입력 구조를 먼저 읽고, 필요한 컬럼이나 selector만 선택한 뒤 결과를 명시적인 변수에 저장한다. `품질 리포트 JSON 저장` 단계에서 중요한 점은 출력 하나를 맞히는 것이 아니라 이후 문제에서 재사용할 수 있는 안정적인 중간 결과를 남기는 것이다. 빈칸에는 파일명, 컬럼명, 키 이름, 반복 제어 중 핵심 위치만 남겼기 때문에 학생이 데이터 구조를 이해해야 완성할 수 있다. 자주 보이는 오답은 컬럼명을 추측하거나, 중복과 실패 상태를 처리하지 않고 바로 저장하는 방식이다.

---

## 문제 12 정답 — SQLite 테이블 생성

~~~python
conn = sqlite3.connect('lesson08_feed.db')
conn.execute('drop table if exists feed')
conn.execute('create table feed(record_id text, title text, category text, price integer, stock integer, status text)')
conn.executemany('insert into feed values(:record_id, :title, :category, :price, :stock, :status)', clean_rows)
print(conn.execute('select count(*) from feed').fetchone()[0])
conn.close()
~~~

### 왜 이 코드가 정답인지

이 답안은 문제에서 요구한 입력 구조를 먼저 읽고, 필요한 컬럼이나 selector만 선택한 뒤 결과를 명시적인 변수에 저장한다. `SQLite 테이블 생성` 단계에서 중요한 점은 출력 하나를 맞히는 것이 아니라 이후 문제에서 재사용할 수 있는 안정적인 중간 결과를 남기는 것이다. 빈칸에는 파일명, 컬럼명, 키 이름, 반복 제어 중 핵심 위치만 남겼기 때문에 학생이 데이터 구조를 이해해야 완성할 수 있다. 자주 보이는 오답은 컬럼명을 추측하거나, 중복과 실패 상태를 처리하지 않고 바로 저장하는 방식이다.

---

## 문제 13 정답 — 가격 높은 재고 상품 조회

~~~python
conn = sqlite3.connect('lesson08_feed.db')
rows_sql = conn.execute('select title, price from feed where stock > 0 and price >= ? order by price desc', (50000,)).fetchall()
print(rows_sql[:5])
conn.close()
~~~

### 왜 이 코드가 정답인지

이 답안은 문제에서 요구한 입력 구조를 먼저 읽고, 필요한 컬럼이나 selector만 선택한 뒤 결과를 명시적인 변수에 저장한다. `가격 높은 재고 상품 조회` 단계에서 중요한 점은 출력 하나를 맞히는 것이 아니라 이후 문제에서 재사용할 수 있는 안정적인 중간 결과를 남기는 것이다. 빈칸에는 파일명, 컬럼명, 키 이름, 반복 제어 중 핵심 위치만 남겼기 때문에 학생이 데이터 구조를 이해해야 완성할 수 있다. 자주 보이는 오답은 컬럼명을 추측하거나, 중복과 실패 상태를 처리하지 않고 바로 저장하는 방식이다.

---

## 문제 14 정답 — HTML 표에서 행 개수 읽기

~~~python
soup = BeautifulSoup(load_text('validation_panel.html'), 'html.parser')
items = soup.select('tbody tr')
print(len(items))
~~~

### 왜 이 코드가 정답인지

이 답안은 문제에서 요구한 입력 구조를 먼저 읽고, 필요한 컬럼이나 selector만 선택한 뒤 결과를 명시적인 변수에 저장한다. `HTML 표에서 행 개수 읽기` 단계에서 중요한 점은 출력 하나를 맞히는 것이 아니라 이후 문제에서 재사용할 수 있는 안정적인 중간 결과를 남기는 것이다. 빈칸에는 파일명, 컬럼명, 키 이름, 반복 제어 중 핵심 위치만 남겼기 때문에 학생이 데이터 구조를 이해해야 완성할 수 있다. 자주 보이는 오답은 컬럼명을 추측하거나, 중복과 실패 상태를 처리하지 않고 바로 저장하는 방식이다.

---

## 문제 15 정답 — 검증 파이프라인 함수 만들기

~~~python
def build_clean_feed():
    rows = load_csv('raw_product_feed.csv')
    rules = load_json('category_rules.json')
    checked = []
    seen = set()
    clean = []
    for row in rows:
        errors = validate_feed_row(row, rules)
        if row['record_id'] in seen:
            errors.append('duplicate:record_id')
        seen.add(row['record_id'])
        checked.append({**row, 'errors': '|'.join(errors), 'is_valid': not errors})
        if not errors:
            clean.append({'record_id': row['record_id'], 'title': row['title'].strip(), 'category': row['category'], 'price': parse_price(row['price_text']), 'stock': parse_stock(row['stock']), 'status': row['status']})
    return checked, clean
checked_rows, clean_rows = build_clean_feed()
print(len(checked_rows), len(clean_rows))
~~~

### 왜 이 코드가 정답인지

이 답안은 문제에서 요구한 입력 구조를 먼저 읽고, 필요한 컬럼이나 selector만 선택한 뒤 결과를 명시적인 변수에 저장한다. `검증 파이프라인 함수 만들기` 단계에서 중요한 점은 출력 하나를 맞히는 것이 아니라 이후 문제에서 재사용할 수 있는 안정적인 중간 결과를 남기는 것이다. 빈칸에는 파일명, 컬럼명, 키 이름, 반복 제어 중 핵심 위치만 남겼기 때문에 학생이 데이터 구조를 이해해야 완성할 수 있다. 자주 보이는 오답은 컬럼명을 추측하거나, 중복과 실패 상태를 처리하지 않고 바로 저장하는 방식이다.

### 보강 설명 1

교사는 학생 답안을 볼 때 값이 우연히 맞았는지, 입력 검증과 저장 순서를 설명할 수 있는지를 함께 확인한다. 특히 자동화 코드는 실패했을 때의 로그가 없으면 운영 코드로 보기 어렵다.

### 보강 설명 2

교사는 학생 답안을 볼 때 값이 우연히 맞았는지, 입력 검증과 저장 순서를 설명할 수 있는지를 함께 확인한다. 특히 자동화 코드는 실패했을 때의 로그가 없으면 운영 코드로 보기 어렵다.

### 보강 설명 3

교사는 학생 답안을 볼 때 값이 우연히 맞았는지, 입력 검증과 저장 순서를 설명할 수 있는지를 함께 확인한다. 특히 자동화 코드는 실패했을 때의 로그가 없으면 운영 코드로 보기 어렵다.

### 보강 설명 4

교사는 학생 답안을 볼 때 값이 우연히 맞았는지, 입력 검증과 저장 순서를 설명할 수 있는지를 함께 확인한다. 특히 자동화 코드는 실패했을 때의 로그가 없으면 운영 코드로 보기 어렵다.

### 보강 설명 5

교사는 학생 답안을 볼 때 값이 우연히 맞았는지, 입력 검증과 저장 순서를 설명할 수 있는지를 함께 확인한다. 특히 자동화 코드는 실패했을 때의 로그가 없으면 운영 코드로 보기 어렵다.

### 보강 설명 6

교사는 학생 답안을 볼 때 값이 우연히 맞았는지, 입력 검증과 저장 순서를 설명할 수 있는지를 함께 확인한다. 특히 자동화 코드는 실패했을 때의 로그가 없으면 운영 코드로 보기 어렵다.

### 보강 설명 7

교사는 학생 답안을 볼 때 값이 우연히 맞았는지, 입력 검증과 저장 순서를 설명할 수 있는지를 함께 확인한다. 특히 자동화 코드는 실패했을 때의 로그가 없으면 운영 코드로 보기 어렵다.

### 보강 설명 8

교사는 학생 답안을 볼 때 값이 우연히 맞았는지, 입력 검증과 저장 순서를 설명할 수 있는지를 함께 확인한다. 특히 자동화 코드는 실패했을 때의 로그가 없으면 운영 코드로 보기 어렵다.

### 보강 설명 9

교사는 학생 답안을 볼 때 값이 우연히 맞았는지, 입력 검증과 저장 순서를 설명할 수 있는지를 함께 확인한다. 특히 자동화 코드는 실패했을 때의 로그가 없으면 운영 코드로 보기 어렵다.

### 보강 설명 10

교사는 학생 답안을 볼 때 값이 우연히 맞았는지, 입력 검증과 저장 순서를 설명할 수 있는지를 함께 확인한다. 특히 자동화 코드는 실패했을 때의 로그가 없으면 운영 코드로 보기 어렵다.

### 보강 설명 11

교사는 학생 답안을 볼 때 값이 우연히 맞았는지, 입력 검증과 저장 순서를 설명할 수 있는지를 함께 확인한다. 특히 자동화 코드는 실패했을 때의 로그가 없으면 운영 코드로 보기 어렵다.

### 보강 설명 12

교사는 학생 답안을 볼 때 값이 우연히 맞았는지, 입력 검증과 저장 순서를 설명할 수 있는지를 함께 확인한다. 특히 자동화 코드는 실패했을 때의 로그가 없으면 운영 코드로 보기 어렵다.

### 보강 설명 13

교사는 학생 답안을 볼 때 값이 우연히 맞았는지, 입력 검증과 저장 순서를 설명할 수 있는지를 함께 확인한다. 특히 자동화 코드는 실패했을 때의 로그가 없으면 운영 코드로 보기 어렵다.

### 보강 설명 14

교사는 학생 답안을 볼 때 값이 우연히 맞았는지, 입력 검증과 저장 순서를 설명할 수 있는지를 함께 확인한다. 특히 자동화 코드는 실패했을 때의 로그가 없으면 운영 코드로 보기 어렵다.

### 보강 설명 15

교사는 학생 답안을 볼 때 값이 우연히 맞았는지, 입력 검증과 저장 순서를 설명할 수 있는지를 함께 확인한다. 특히 자동화 코드는 실패했을 때의 로그가 없으면 운영 코드로 보기 어렵다.

### 보강 설명 16

교사는 학생 답안을 볼 때 값이 우연히 맞았는지, 입력 검증과 저장 순서를 설명할 수 있는지를 함께 확인한다. 특히 자동화 코드는 실패했을 때의 로그가 없으면 운영 코드로 보기 어렵다.

### 보강 설명 17

교사는 학생 답안을 볼 때 값이 우연히 맞았는지, 입력 검증과 저장 순서를 설명할 수 있는지를 함께 확인한다. 특히 자동화 코드는 실패했을 때의 로그가 없으면 운영 코드로 보기 어렵다.

### 보강 설명 18

교사는 학생 답안을 볼 때 값이 우연히 맞았는지, 입력 검증과 저장 순서를 설명할 수 있는지를 함께 확인한다. 특히 자동화 코드는 실패했을 때의 로그가 없으면 운영 코드로 보기 어렵다.

### 보강 설명 19

교사는 학생 답안을 볼 때 값이 우연히 맞았는지, 입력 검증과 저장 순서를 설명할 수 있는지를 함께 확인한다. 특히 자동화 코드는 실패했을 때의 로그가 없으면 운영 코드로 보기 어렵다.

### 보강 설명 20

교사는 학생 답안을 볼 때 값이 우연히 맞았는지, 입력 검증과 저장 순서를 설명할 수 있는지를 함께 확인한다. 특히 자동화 코드는 실패했을 때의 로그가 없으면 운영 코드로 보기 어렵다.

### 보강 설명 21

교사는 학생 답안을 볼 때 값이 우연히 맞았는지, 입력 검증과 저장 순서를 설명할 수 있는지를 함께 확인한다. 특히 자동화 코드는 실패했을 때의 로그가 없으면 운영 코드로 보기 어렵다.

### 보강 설명 22

교사는 학생 답안을 볼 때 값이 우연히 맞았는지, 입력 검증과 저장 순서를 설명할 수 있는지를 함께 확인한다. 특히 자동화 코드는 실패했을 때의 로그가 없으면 운영 코드로 보기 어렵다.

### 보강 설명 23

교사는 학생 답안을 볼 때 값이 우연히 맞았는지, 입력 검증과 저장 순서를 설명할 수 있는지를 함께 확인한다. 특히 자동화 코드는 실패했을 때의 로그가 없으면 운영 코드로 보기 어렵다.
