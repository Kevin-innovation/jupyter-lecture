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

# 레슨 08 — 실습 문제

수집 데이터 검증과 저장 레슨의 학생용 문제 노트북이다. 강의 노트북을 먼저 실행한 뒤 빈칸을 직접 채운다.

## 통과 기준

- 총 15문제 중 12문제 이상 정상 출력이면 통과.
- 문제 1~5는 구조 확인, 6~10은 응용 처리, 11~15는 저장과 운영 요약이다.
- 정답값은 적지 않는다. 출력 형태와 fixture 구조를 보고 직접 판단한다.

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

## 문제 1 — 원본 피드 행 수 확인

지시된 값을 코드로 추출하거나 상태를 변경한다. 실행 결과는 한 줄 출력, 리스트, 딕셔너리, 저장 파일 생성 중 하나다.

**기대 결과 형태**: 요구한 값이 출력되거나 지정한 파일이 생성된다.

**빈칸 힌트**: 파일 이름과 변수명을 채운다. 완성 코드를 그대로 따라 쓰지 말고 `____` 부분만 스스로 판단한다.

~~~python
rows = load_csv('____')
print(len(____))
~~~

---

## 문제 2 — 검증 규칙 JSON 읽기

지시된 값을 코드로 추출하거나 상태를 변경한다. 실행 결과는 한 줄 출력, 리스트, 딕셔너리, 저장 파일 생성 중 하나다.

**기대 결과 형태**: 요구한 값이 출력되거나 지정한 파일이 생성된다.

**빈칸 힌트**: 규칙 파일과 필수 컬럼 키를 찾는다. 완성 코드를 그대로 따라 쓰지 말고 `____` 부분만 스스로 판단한다.

~~~python
rules = load_json('____')
print(rules['____'])
~~~

---

## 문제 3 — 가격 문자열을 숫자로 변환

지시된 값을 코드로 추출하거나 상태를 변경한다. 실행 결과는 한 줄 출력, 리스트, 딕셔너리, 저장 파일 생성 중 하나다.

**기대 결과 형태**: 요구한 값이 출력되거나 지정한 파일이 생성된다.

**빈칸 힌트**: 가격 컬럼 이름을 확인한다. 완성 코드를 그대로 따라 쓰지 말고 `____` 부분만 스스로 판단한다.

~~~python
price = parse_price(rows[0]['____'])
print(____)
~~~

---

## 문제 4 — 재고 값을 정수로 변환

지시된 값을 코드로 추출하거나 상태를 변경한다. 실행 결과는 한 줄 출력, 리스트, 딕셔너리, 저장 파일 생성 중 하나다.

**기대 결과 형태**: 요구한 값이 출력되거나 지정한 파일이 생성된다.

**빈칸 힌트**: 재고 컬럼 이름을 확인한다. 완성 코드를 그대로 따라 쓰지 말고 `____` 부분만 스스로 판단한다.

~~~python
stock = parse_stock(rows[0]['____'])
print(____)
~~~

---

## 문제 5 — 첫 행 오류 코드 확인

지시된 값을 코드로 추출하거나 상태를 변경한다. 실행 결과는 한 줄 출력, 리스트, 딕셔너리, 저장 파일 생성 중 하나다.

**기대 결과 형태**: 요구한 값이 출력되거나 지정한 파일이 생성된다.

**빈칸 힌트**: 검증 함수에는 행과 규칙이 모두 필요하다. 완성 코드를 그대로 따라 쓰지 말고 `____` 부분만 스스로 판단한다.

~~~python
errors = validate_feed_row(rows[0], ____)
print(____)
~~~

---

## 문제 6 — 전체 행 검증 결과 만들기

지시된 값을 코드로 추출하거나 상태를 변경한다. 실행 결과는 한 줄 출력, 리스트, 딕셔너리, 저장 파일 생성 중 하나다.

**기대 결과 형태**: 요구한 값이 출력되거나 지정한 파일이 생성된다.

**빈칸 힌트**: 오류 목록이 비어 있으면 유효하다. 완성 코드를 그대로 따라 쓰지 말고 `____` 부분만 스스로 판단한다.

~~~python
checked = []
for row in rows:
    errors = validate_feed_row(row, rules)
    checked.append({**row, 'errors': '|'.join(errors), 'is_valid': not ____})
print(len(____))
~~~

---

## 문제 7 — 중복 record_id 찾기

지시된 값을 코드로 추출하거나 상태를 변경한다. 실행 결과는 한 줄 출력, 리스트, 딕셔너리, 저장 파일 생성 중 하나다.

**기대 결과 형태**: 요구한 값이 출력되거나 지정한 파일이 생성된다.

**빈칸 힌트**: 식별자 컬럼과 중복 목록에 넣을 값을 채운다. 완성 코드를 그대로 따라 쓰지 말고 `____` 부분만 스스로 판단한다.

~~~python
seen = set()
duplicates = []
for row in checked:
    key = row['____']
    if key in seen:
        duplicates.append(____)
    seen.add(key)
print(duplicates[:5])
~~~

---

## 문제 8 — 유효 행만 정제 리스트로 만들기

지시된 값을 코드로 추출하거나 상태를 변경한다. 실행 결과는 한 줄 출력, 리스트, 딕셔너리, 저장 파일 생성 중 하나다.

**기대 결과 형태**: 요구한 값이 출력되거나 지정한 파일이 생성된다.

**빈칸 힌트**: 검증 결과의 boolean 컬럼을 사용한다. 완성 코드를 그대로 따라 쓰지 말고 `____` 부분만 스스로 판단한다.

~~~python
clean_rows = []
seen = set()
for row in checked:
    if row['record_id'] in seen:
        continue
    seen.add(row['record_id'])
    if row['____']:
        clean_rows.append({'record_id': row['record_id'], 'title': row['title'].strip(), 'category': row['category'], 'price': parse_price(row['price_text']), 'stock': parse_stock(row['stock']), 'status': row['status']})
print(len(clean_rows))
~~~

---

## 문제 9 — 카테고리별 개수 요약

지시된 값을 코드로 추출하거나 상태를 변경한다. 실행 결과는 한 줄 출력, 리스트, 딕셔너리, 저장 파일 생성 중 하나다.

**기대 결과 형태**: 요구한 값이 출력되거나 지정한 파일이 생성된다.

**빈칸 힌트**: 요약 기준 컬럼은 category다. 완성 코드를 그대로 따라 쓰지 말고 `____` 부분만 스스로 판단한다.

~~~python
summary = {}
for row in clean_rows:
    summary[row['____']] = summary.get(row['category'], 0) + 1
print(summary)
~~~

---

## 문제 10 — 정제 CSV 저장

지시된 값을 코드로 추출하거나 상태를 변경한다. 실행 결과는 한 줄 출력, 리스트, 딕셔너리, 저장 파일 생성 중 하나다.

**기대 결과 형태**: 요구한 값이 출력되거나 지정한 파일이 생성된다.

**빈칸 힌트**: 저장할 행 목록과 파일 존재 확인 메서드를 채운다. 완성 코드를 그대로 따라 쓰지 말고 `____` 부분만 스스로 판단한다.

~~~python
write_csv('lesson08_clean_feed.csv', ____)
print(Path('lesson08_clean_feed.csv').____())
~~~

---

## 문제 11 — 품질 리포트 JSON 저장

지시된 값을 코드로 추출하거나 상태를 변경한다. 실행 결과는 한 줄 출력, 리스트, 딕셔너리, 저장 파일 생성 중 하나다.

**기대 결과 형태**: 요구한 값이 출력되거나 지정한 파일이 생성된다.

**빈칸 힌트**: 중복 목록 변수명을 채운다. 완성 코드를 그대로 따라 쓰지 말고 `____` 부분만 스스로 판단한다.

~~~python
report = {'raw_count': len(rows), 'clean_count': len(clean_rows), 'duplicate_count': len(____)}
Path('lesson08_quality_report.json').write_text(json.dumps(report, ensure_ascii=False, indent=2), encoding='utf-8')
print(report)
~~~

---

## 문제 12 — SQLite 테이블 생성

지시된 값을 코드로 추출하거나 상태를 변경한다. 실행 결과는 한 줄 출력, 리스트, 딕셔너리, 저장 파일 생성 중 하나다.

**기대 결과 형태**: 요구한 값이 출력되거나 지정한 파일이 생성된다.

**빈칸 힌트**: 삽입할 정제 행 목록을 넣는다. 완성 코드를 그대로 따라 쓰지 말고 `____` 부분만 스스로 판단한다.

~~~python
conn = sqlite3.connect('lesson08_feed.db')
conn.execute('drop table if exists feed')
conn.execute('create table feed(record_id text, title text, category text, price integer, stock integer, status text)')
conn.executemany('insert into feed values(:record_id, :title, :category, :price, :stock, :status)', ____)
print(conn.execute('select count(*) from feed').fetchone()[0])
conn.close()
~~~

---

## 문제 13 — 가격 높은 재고 상품 조회

지시된 값을 코드로 추출하거나 상태를 변경한다. 실행 결과는 한 줄 출력, 리스트, 딕셔너리, 저장 파일 생성 중 하나다.

**기대 결과 형태**: 요구한 값이 출력되거나 지정한 파일이 생성된다.

**빈칸 힌트**: 기준 가격 숫자를 넣는다. 완성 코드를 그대로 따라 쓰지 말고 `____` 부분만 스스로 판단한다.

~~~python
conn = sqlite3.connect('lesson08_feed.db')
rows_sql = conn.execute('select title, price from feed where stock > 0 and price >= ? order by price desc', (____,)).fetchall()
print(rows_sql[:5])
conn.close()
~~~

---

## 문제 14 — HTML 표에서 행 개수 읽기

지시된 값을 코드로 추출하거나 상태를 변경한다. 실행 결과는 한 줄 출력, 리스트, 딕셔너리, 저장 파일 생성 중 하나다.

**기대 결과 형태**: 요구한 값이 출력되거나 지정한 파일이 생성된다.

**빈칸 힌트**: 검증 패널 파일과 table row selector를 채운다. 완성 코드를 그대로 따라 쓰지 말고 `____` 부분만 스스로 판단한다.

~~~python
soup = BeautifulSoup(load_text('____'), 'html.parser')
items = soup.select('____')
print(len(items))
~~~

---

## 문제 15 — 검증 파이프라인 함수 만들기

지시된 값을 코드로 추출하거나 상태를 변경한다. 실행 결과는 한 줄 출력, 리스트, 딕셔너리, 저장 파일 생성 중 하나다.

**기대 결과 형태**: 요구한 값이 출력되거나 지정한 파일이 생성된다.

**빈칸 힌트**: 검증 결과와 정제 결과를 함께 반환한다. 완성 코드를 그대로 따라 쓰지 말고 `____` 부분만 스스로 판단한다.

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
    return ____, ____
checked_rows, clean_rows = build_clean_feed()
print(len(checked_rows), len(clean_rows))
~~~

### 보강 설명 1

문제가 막히면 HTML 태그, CSV 헤더, JSON 키를 먼저 소리 내어 읽는다. 함수명을 떠올리기 전에 입력 데이터의 구조를 확인하면 빈칸을 더 안정적으로 채울 수 있다.

### 보강 설명 2

문제가 막히면 HTML 태그, CSV 헤더, JSON 키를 먼저 소리 내어 읽는다. 함수명을 떠올리기 전에 입력 데이터의 구조를 확인하면 빈칸을 더 안정적으로 채울 수 있다.

### 보강 설명 3

문제가 막히면 HTML 태그, CSV 헤더, JSON 키를 먼저 소리 내어 읽는다. 함수명을 떠올리기 전에 입력 데이터의 구조를 확인하면 빈칸을 더 안정적으로 채울 수 있다.
