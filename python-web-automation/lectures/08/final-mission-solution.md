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

# 레슨 08 — 최종 미션 모범 답안

> 교사 확인용 모범 답안이다. 학생에게는 최종 미션 조건만 제공한다.

## 실행 코드

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


rows = load_csv('raw_product_feed.csv')
print(len(rows))

rules = load_json('category_rules.json')
print(rules['required_fields'])

price = parse_price(rows[0]['price_text'])
print(price)

stock = parse_stock(rows[0]['stock'])
print(stock)

errors = validate_feed_row(rows[0], rules)
print(errors)

checked = []
for row in rows:
    errors = validate_feed_row(row, rules)
    checked.append({**row, 'errors': '|'.join(errors), 'is_valid': not errors})
print(len(checked))

seen = set()
duplicates = []
for row in checked:
    key = row['record_id']
    if key in seen:
        duplicates.append(key)
    seen.add(key)
print(duplicates[:5])

clean_rows = []
seen = set()
for row in checked:
    if row['record_id'] in seen:
        continue
    seen.add(row['record_id'])
    if row['is_valid']:
        clean_rows.append({'record_id': row['record_id'], 'title': row['title'].strip(), 'category': row['category'], 'price': parse_price(row['price_text']), 'stock': parse_stock(row['stock']), 'status': row['status']})
print(len(clean_rows))

summary = {}
for row in clean_rows:
    summary[row['category']] = summary.get(row['category'], 0) + 1
print(summary)

write_csv('lesson08_clean_feed.csv', clean_rows)
print(Path('lesson08_clean_feed.csv').exists())

report = {'raw_count': len(rows), 'clean_count': len(clean_rows), 'duplicate_count': len(duplicates)}
Path('lesson08_quality_report.json').write_text(json.dumps(report, ensure_ascii=False, indent=2), encoding='utf-8')
print(report)

conn = sqlite3.connect('lesson08_feed.db')
conn.execute('drop table if exists feed')
conn.execute('create table feed(record_id text, title text, category text, price integer, stock integer, status text)')
conn.executemany('insert into feed values(:record_id, :title, :category, :price, :stock, :status)', clean_rows)
print(conn.execute('select count(*) from feed').fetchone()[0])
conn.close()

conn = sqlite3.connect('lesson08_feed.db')
rows_sql = conn.execute('select title, price from feed where stock > 0 and price >= ? order by price desc', (50000,)).fetchall()
print(rows_sql[:5])
conn.close()

soup = BeautifulSoup(load_text('validation_panel.html'), 'html.parser')
items = soup.select('tbody tr')
print(len(items))

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

## 채점 메모

- 입력 파일을 모두 읽었는지 확인한다.
- 검증 기준이 코드에 명시되어 있는지 확인한다.
- 저장 파일과 운영 요약이 함께 있는지 확인한다.
