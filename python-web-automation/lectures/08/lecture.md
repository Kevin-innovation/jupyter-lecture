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

이 셀은 원본 피드를 먼저 읽고 검증 규칙과 연결하는 출발점이다. 학생은 행 수와 필수 컬럼을 확인하며 저장 전에 무엇을 검사해야 하는지 정리한다.

~~~python
rows = load_csv('raw_product_feed.csv')
rules = load_json('category_rules.json')
print(len(rows), rules['required_fields'])
~~~
이 단계 이후에는 원본 행과 정제 행이 분리된다. 학생에게 어떤 행이 제외되었는지 말로 설명하게 한다.

---

## 4. 자료 구조 확인

가격과 재고를 숫자로 바꾸는 과정은 데이터 정제의 기본이다. 출력값을 맞히는 것보다 변환 함수가 실패 값을 어떻게 다루는지 관찰하게 한다.

~~~python
sample = rows[0]
print(sample)
print(parse_price(sample['price_text']), parse_stock(sample['stock']))
~~~
변환 함수는 실패 값을 조용히 숨기지 않아야 한다. 이상한 입력이 들어왔을 때 어떤 결과가 나오는지 확인한다.

---

## 5. 품질 기준 적용

검증 결과를 행마다 붙이면 오류를 숨기지 않고 추적할 수 있다. 학생은 errors와 is_valid가 서로 어떤 관계인지 설명해야 한다.

~~~python
checked = []
for row in rows:
    errors = validate_feed_row(row, rules)
    checked.append({**row, 'errors': '|'.join(errors), 'is_valid': not errors})
print(checked[0])
~~~
검증 결과 컬럼은 피드백의 근거가 된다. 교사는 errors 값을 보고 학생의 기준 적용 여부를 빠르게 판단할 수 있다.

---

## 6. 저장과 보고

중복 제거는 저장 직전에 반드시 필요한 단계다. record_id를 기준으로 보되 실제 운영에서는 URL이나 제목까지 묶을 수 있음을 함께 다룬다.

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
중복 목록은 버리는 데이터가 아니라 품질 리포트의 일부다. 운영자는 중복이 왜 생겼는지 알아야 한다.

---

## 7. 운영 관점 점검

정제 행을 만들 때는 필요한 필드만 남긴다. 원본 문자열과 저장용 숫자를 구분하면 이후 집계와 검색이 쉬워진다.

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
정제 리스트는 저장 가능한 최소 구조다. 원본 HTML이나 임시 문자열을 그대로 끌고 가지 않는 점이 중요하다.

---

## 8. 마무리 체크

CSV와 JSON 저장은 서로 다른 목적을 가진다. CSV는 상세 목록을 확인하고 JSON은 품질 요약을 공유하는 데 적합하다.

~~~python
write_csv('lesson08_clean_feed.csv', clean_rows)
report = {'raw_count': len(rows), 'clean_count': len(clean_rows), 'duplicate_count': len(duplicates)}
Path('lesson08_quality_report.json').write_text(json.dumps(report, ensure_ascii=False, indent=2), encoding='utf-8')
print(report)
~~~
저장 파일은 결과 확인용 산출물이다. 파일 존재만 보지 말고 내용 행 수와 헤더를 함께 확인한다.

---

## 9. 핵심 개념

SQLite 예제는 작은 데이터라도 질의 가능한 구조로 저장할 수 있음을 보여준다. 학생에게 category별 집계가 왜 편해지는지 묻는다.

~~~python
conn = sqlite3.connect('lesson08_feed.db')
conn.execute('drop table if exists feed')
conn.execute('create table feed(record_id text, title text, category text, price integer, stock integer, status text)')
conn.executemany('insert into feed values(:record_id, :title, :category, :price, :stock, :status)', clean_rows)
print(conn.execute('select category, count(*) from feed group by category').fetchall())
conn.close()
~~~
데이터베이스 조회는 저장 후 활용 가능성을 보여준다. 단순 파일 저장보다 질문을 던질 수 있는 구조가 된다.

---

## 데이터 출처와 안전 규칙

raw_product_feed.csv는 의도적으로 누락, 중복, 형식 오류가 섞인 합성 상품 피드다. category_rules.json은 허용 카테고리, 상태, 최대 가격 기준을 담는다. validation_panel.html은 같은 데이터를 웹 표 형태로 확인하는 연습용 페이지다. schema_notes.txt는 사람이 읽는 스키마 설명이다.

- 모든 파일은 수업용 합성 데이터다.
- 실제 사이트에 반복 요청하지 않는다.
- 저장 파일은 레슨 폴더 또는 코랩 현재 작업 폴더에만 만든다.
- 외부 사이트로 확장할 때는 약관, robots.txt, 요청 간격, 개인정보 포함 여부를 먼저 확인한다.

---

## 강의 보강 노트

이 절은 수업 중 교사가 질문으로 풀어낼 수 있는 운영형 설명이다. 학생이 셀을 실행한 뒤 결과만 맞히지 않고 자동화 절차를 말로 설명하도록 돕는다.

### 1. 검증 우선 원칙

수집 데이터는 저장되기 전까지 임시 결과로 봐야 한다. 필수 컬럼, 값 범위, 허용 category, status를 확인한 뒤에야 운영자가 믿을 수 있는 데이터가 된다.

### 2. 오류 코드 설계

오류를 `invalid` 하나로 뭉치면 나중에 어떤 기준이 문제였는지 알 수 없다. `missing:title`, `invalid:price`처럼 구체적인 코드로 남기면 학생과 교사 모두 피드백이 빠르다.

### 3. 중복 기준

중복은 행 전체가 같은지보다 업무상 같은 항목인지가 중요하다. 이 fixture에서는 record_id를 기준으로 하지만 실제 운영에서는 URL, 제목, 날짜를 함께 묶어 key를 만들 수도 있다.

### 4. 정제 행과 오류 행 분리

유효한 행만 저장하는 것과 오류 행을 버리는 것은 다르다. 오류 행도 별도 리포트에 남겨야 다음 수집 기준을 조정할 수 있다는 점을 강조한다.

### 5. CSV와 JSON의 역할

CSV는 표로 확인하기 좋고 JSON은 요약과 설정을 담기 좋다. 학생에게 두 형식의 장단점을 비교하게 하면 저장 파일을 무작정 하나로 고르지 않게 된다.

### 6. SQLite 도입 이유

작은 수업 데이터라도 SQL로 조회해 보면 필터링과 집계가 왜 필요한지 이해하기 쉽다. 데이터베이스는 큰 시스템이 아니라 구조화된 질문을 던지는 도구로 소개한다.

### 7. 운영 리포트

최종 결과에는 raw_count, clean_count, duplicate_count가 함께 있어야 한다. 숫자 차이를 설명할 수 있어야 수집기가 실제 운영에서 신뢰를 얻는다.

### 체크포인트 1: 중간 변수 되돌아본다

운영자가 결과를 이해할 수 있게, 레슨 08 수집 데이터 검증과 저장에서는 중간 변수을/를 되돌아본다는 과정을 별도의 단계로 둔다. 이때 학생은 화면에 보이는 값 하나보다 어떤 입력에서 어떤 변환을 거쳐 결과가 나왔는지 설명해야 한다. 강사는 변수명, 출력 형태, 저장 여부를 함께 확인하며 다음 실행에서도 같은 결과가 나오는지 묻는다.

### 체크포인트 2: 입력 파일 표준화한다

재실행했을 때 같은 결과를 얻기 위해, 레슨 08 수집 데이터 검증과 저장에서는 입력 파일을/를 표준화한다는 과정을 별도의 단계로 둔다. 이때 학생은 화면에 보이는 값 하나보다 어떤 입력에서 어떤 변환을 거쳐 결과가 나왔는지 설명해야 한다. 강사는 변수명, 출력 형태, 저장 여부를 함께 확인하며 다음 실행에서도 같은 결과가 나오는지 묻는다.

### 체크포인트 3: 선택자 검증한다

다음 셀에서 재사용하기 위해, 레슨 08 수집 데이터 검증과 저장에서는 선택자을/를 검증한다는 과정을 별도의 단계로 둔다. 이때 학생은 화면에 보이는 값 하나보다 어떤 입력에서 어떤 변환을 거쳐 결과가 나왔는지 설명해야 한다. 강사는 변수명, 출력 형태, 저장 여부를 함께 확인하며 다음 실행에서도 같은 결과가 나오는지 묻는다.

### 체크포인트 4: 오류 메시지 확인한다

실제 사이트 확장 전에 위험을 낮추기 위해, 레슨 08 수집 데이터 검증과 저장에서는 오류 메시지을/를 확인한다는 과정을 별도의 단계로 둔다. 이때 학생은 화면에 보이는 값 하나보다 어떤 입력에서 어떤 변환을 거쳐 결과가 나왔는지 설명해야 한다. 강사는 변수명, 출력 형태, 저장 여부를 함께 확인하며 다음 실행에서도 같은 결과가 나오는지 묻는다.

### 체크포인트 5: 결과 행 수 분리한다

저장 파일의 신뢰도를 높이기 위해, 레슨 08 수집 데이터 검증과 저장에서는 결과 행 수을/를 분리한다는 과정을 별도의 단계로 둔다. 이때 학생은 화면에 보이는 값 하나보다 어떤 입력에서 어떤 변환을 거쳐 결과가 나왔는지 설명해야 한다. 강사는 변수명, 출력 형태, 저장 여부를 함께 확인하며 다음 실행에서도 같은 결과가 나오는지 묻는다.

### 체크포인트 6: 검증 기준 기록한다

코랩과 로컬 실행 차이를 줄이기 위해, 레슨 08 수집 데이터 검증과 저장에서는 검증 기준을/를 기록한다는 과정을 별도의 단계로 둔다. 이때 학생은 화면에 보이는 값 하나보다 어떤 입력에서 어떤 변환을 거쳐 결과가 나왔는지 설명해야 한다. 강사는 변수명, 출력 형태, 저장 여부를 함께 확인하며 다음 실행에서도 같은 결과가 나오는지 묻는다.

### 체크포인트 7: 중간 변수 비교한다

수업 중 피드백 시간을 줄이기 위해, 레슨 08 수집 데이터 검증과 저장에서는 중간 변수을/를 비교한다는 과정을 별도의 단계로 둔다. 이때 학생은 화면에 보이는 값 하나보다 어떤 입력에서 어떤 변환을 거쳐 결과가 나왔는지 설명해야 한다. 강사는 변수명, 출력 형태, 저장 여부를 함께 확인하며 다음 실행에서도 같은 결과가 나오는지 묻는다.

### 체크포인트 8: 입력 파일 요약한다

학생이 막힌 지점을 찾기 위해, 레슨 08 수집 데이터 검증과 저장에서는 입력 파일을/를 요약한다는 과정을 별도의 단계로 둔다. 이때 학생은 화면에 보이는 값 하나보다 어떤 입력에서 어떤 변환을 거쳐 결과가 나왔는지 설명해야 한다. 강사는 변수명, 출력 형태, 저장 여부를 함께 확인하며 다음 실행에서도 같은 결과가 나오는지 묻는다.

### 체크포인트 9: 선택자 검토한다

운영자가 결과를 이해할 수 있게, 레슨 08 수집 데이터 검증과 저장에서는 선택자을/를 검토한다는 과정을 별도의 단계로 둔다. 이때 학생은 화면에 보이는 값 하나보다 어떤 입력에서 어떤 변환을 거쳐 결과가 나왔는지 설명해야 한다. 강사는 변수명, 출력 형태, 저장 여부를 함께 확인하며 다음 실행에서도 같은 결과가 나오는지 묻는다.

### 체크포인트 10: 오류 메시지 정리한다

재실행했을 때 같은 결과를 얻기 위해, 레슨 08 수집 데이터 검증과 저장에서는 오류 메시지을/를 정리한다는 과정을 별도의 단계로 둔다. 이때 학생은 화면에 보이는 값 하나보다 어떤 입력에서 어떤 변환을 거쳐 결과가 나왔는지 설명해야 한다. 강사는 변수명, 출력 형태, 저장 여부를 함께 확인하며 다음 실행에서도 같은 결과가 나오는지 묻는다.

### 체크포인트 11: 결과 행 수 설명한다

다음 셀에서 재사용하기 위해, 레슨 08 수집 데이터 검증과 저장에서는 결과 행 수을/를 설명한다는 과정을 별도의 단계로 둔다. 이때 학생은 화면에 보이는 값 하나보다 어떤 입력에서 어떤 변환을 거쳐 결과가 나왔는지 설명해야 한다. 강사는 변수명, 출력 형태, 저장 여부를 함께 확인하며 다음 실행에서도 같은 결과가 나오는지 묻는다.

### 체크포인트 12: 검증 기준 저장한다

실제 사이트 확장 전에 위험을 낮추기 위해, 레슨 08 수집 데이터 검증과 저장에서는 검증 기준을/를 저장한다는 과정을 별도의 단계로 둔다. 이때 학생은 화면에 보이는 값 하나보다 어떤 입력에서 어떤 변환을 거쳐 결과가 나왔는지 설명해야 한다. 강사는 변수명, 출력 형태, 저장 여부를 함께 확인하며 다음 실행에서도 같은 결과가 나오는지 묻는다.

### 체크포인트 13: 중간 변수 되돌아본다

저장 파일의 신뢰도를 높이기 위해, 레슨 08 수집 데이터 검증과 저장에서는 중간 변수을/를 되돌아본다는 과정을 별도의 단계로 둔다. 이때 학생은 화면에 보이는 값 하나보다 어떤 입력에서 어떤 변환을 거쳐 결과가 나왔는지 설명해야 한다. 강사는 변수명, 출력 형태, 저장 여부를 함께 확인하며 다음 실행에서도 같은 결과가 나오는지 묻는다.

### 체크포인트 14: 입력 파일 표준화한다

코랩과 로컬 실행 차이를 줄이기 위해, 레슨 08 수집 데이터 검증과 저장에서는 입력 파일을/를 표준화한다는 과정을 별도의 단계로 둔다. 이때 학생은 화면에 보이는 값 하나보다 어떤 입력에서 어떤 변환을 거쳐 결과가 나왔는지 설명해야 한다. 강사는 변수명, 출력 형태, 저장 여부를 함께 확인하며 다음 실행에서도 같은 결과가 나오는지 묻는다.

### 체크포인트 15: 선택자 검증한다

수업 중 피드백 시간을 줄이기 위해, 레슨 08 수집 데이터 검증과 저장에서는 선택자을/를 검증한다는 과정을 별도의 단계로 둔다. 이때 학생은 화면에 보이는 값 하나보다 어떤 입력에서 어떤 변환을 거쳐 결과가 나왔는지 설명해야 한다. 강사는 변수명, 출력 형태, 저장 여부를 함께 확인하며 다음 실행에서도 같은 결과가 나오는지 묻는다.

### 체크포인트 16: 오류 메시지 확인한다

학생이 막힌 지점을 찾기 위해, 레슨 08 수집 데이터 검증과 저장에서는 오류 메시지을/를 확인한다는 과정을 별도의 단계로 둔다. 이때 학생은 화면에 보이는 값 하나보다 어떤 입력에서 어떤 변환을 거쳐 결과가 나왔는지 설명해야 한다. 강사는 변수명, 출력 형태, 저장 여부를 함께 확인하며 다음 실행에서도 같은 결과가 나오는지 묻는다.

### 체크포인트 17: 결과 행 수 분리한다

운영자가 결과를 이해할 수 있게, 레슨 08 수집 데이터 검증과 저장에서는 결과 행 수을/를 분리한다는 과정을 별도의 단계로 둔다. 이때 학생은 화면에 보이는 값 하나보다 어떤 입력에서 어떤 변환을 거쳐 결과가 나왔는지 설명해야 한다. 강사는 변수명, 출력 형태, 저장 여부를 함께 확인하며 다음 실행에서도 같은 결과가 나오는지 묻는다.

### 체크포인트 18: 검증 기준 기록한다

재실행했을 때 같은 결과를 얻기 위해, 레슨 08 수집 데이터 검증과 저장에서는 검증 기준을/를 기록한다는 과정을 별도의 단계로 둔다. 이때 학생은 화면에 보이는 값 하나보다 어떤 입력에서 어떤 변환을 거쳐 결과가 나왔는지 설명해야 한다. 강사는 변수명, 출력 형태, 저장 여부를 함께 확인하며 다음 실행에서도 같은 결과가 나오는지 묻는다.

### 체크포인트 19: 중간 변수 비교한다

다음 셀에서 재사용하기 위해, 레슨 08 수집 데이터 검증과 저장에서는 중간 변수을/를 비교한다는 과정을 별도의 단계로 둔다. 이때 학생은 화면에 보이는 값 하나보다 어떤 입력에서 어떤 변환을 거쳐 결과가 나왔는지 설명해야 한다. 강사는 변수명, 출력 형태, 저장 여부를 함께 확인하며 다음 실행에서도 같은 결과가 나오는지 묻는다.

### 체크포인트 20: 입력 파일 요약한다

실제 사이트 확장 전에 위험을 낮추기 위해, 레슨 08 수집 데이터 검증과 저장에서는 입력 파일을/를 요약한다는 과정을 별도의 단계로 둔다. 이때 학생은 화면에 보이는 값 하나보다 어떤 입력에서 어떤 변환을 거쳐 결과가 나왔는지 설명해야 한다. 강사는 변수명, 출력 형태, 저장 여부를 함께 확인하며 다음 실행에서도 같은 결과가 나오는지 묻는다.

### 체크포인트 21: 선택자 검토한다

저장 파일의 신뢰도를 높이기 위해, 레슨 08 수집 데이터 검증과 저장에서는 선택자을/를 검토한다는 과정을 별도의 단계로 둔다. 이때 학생은 화면에 보이는 값 하나보다 어떤 입력에서 어떤 변환을 거쳐 결과가 나왔는지 설명해야 한다. 강사는 변수명, 출력 형태, 저장 여부를 함께 확인하며 다음 실행에서도 같은 결과가 나오는지 묻는다.

### 체크포인트 22: 오류 메시지 정리한다

코랩과 로컬 실행 차이를 줄이기 위해, 레슨 08 수집 데이터 검증과 저장에서는 오류 메시지을/를 정리한다는 과정을 별도의 단계로 둔다. 이때 학생은 화면에 보이는 값 하나보다 어떤 입력에서 어떤 변환을 거쳐 결과가 나왔는지 설명해야 한다. 강사는 변수명, 출력 형태, 저장 여부를 함께 확인하며 다음 실행에서도 같은 결과가 나오는지 묻는다.
