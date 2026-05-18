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

[![Open in Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/Kevin-innovation/jupyter-lecture/blob/main/python-web-automation/lectures/08/%5B%ED%95%99%EC%83%9D%EC%9A%A9%5D%20%EB%A0%88%EC%8A%A8%2008%20%E2%80%94%20%EC%88%98%EC%A7%91%20%EB%8D%B0%EC%9D%B4%ED%84%B0%20%EA%B2%80%EC%A6%9D%EA%B3%BC%20%EC%A0%80%EC%9E%A5.ipynb)

> 코랩에서 실행하려면 우측 상단 또는 아래의 **Open in Colab** 버튼을 클릭한다. 이 노트북은 학생용 읽기와 따라하기 자료다.

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

## 수업 운영 메모

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

---

## 저장 전 검증 흐름 정리

이번 레슨의 핵심은 저장 버튼을 누르기 전에 데이터가 운영 기준을 통과했는지 확인하는 것이다. 수집 자동화는 화면에서 값을 가져오는 순간보다 저장 직전이 더 중요하다. 저장된 파일은 이후 리포트, 메시지, 대시보드에 재사용되므로 잘못된 행 하나가 여러 화면으로 퍼질 수 있다.

수업에서는 다음 순서를 반복해서 확인한다.

1. 원본 행을 그대로 보관한다.
2. 필수 컬럼과 값 범위를 검사한다.
3. 오류 코드를 행마다 남긴다.
4. 중복 기준을 별도로 적용한다.
5. 정제 행과 오류 리포트를 분리 저장한다.
6. 저장 후 행 수와 요약 값을 다시 확인한다.

학생이 흔히 하는 실수는 오류가 있는 행을 조용히 건너뛰는 것이다. 수업 중에는 “왜 제외되었는지”를 반드시 말하게 한다. 예를 들어 제목이 비었으면 missing:title, 가격이 숫자로 바뀌지 않으면 invalid:price, 같은 record_id가 다시 나오면 duplicate:record_id로 남긴다. 이렇게 남겨야 나중에 원본 페이지 구조가 바뀌었는지, 수집 규칙이 너무 엄격한지 판단할 수 있다.

## CSV, JSON, SQLite 선택 기준

CSV는 사람이 표로 열어 확인하기 쉽고, 학부모 피드백이나 운영 점검처럼 행 단위 검토가 필요한 상황에 적합하다. JSON은 요약 값, 설정값, 오류 통계처럼 구조가 있는 데이터를 저장하기 좋다. SQLite는 저장한 뒤 다시 질문해야 할 때 쓴다. 예를 들어 카테고리별 개수, 재고가 남은 항목, 특정 가격 이상 항목을 반복해서 조회할 때는 SQL이 더 안정적이다.

이번 수업에서는 세 가지 저장 방식을 모두 경험하지만, 학생에게 한 가지 정답을 강요하지 않는다. 대신 어떤 산출물이 누구에게 전달되는지에 따라 형식을 고르게 한다. 운영자가 엑셀로 열어야 하면 CSV, 자동화 다음 단계가 읽어야 하면 JSON, 여러 조건으로 다시 조회해야 하면 SQLite가 자연스럽다.

## 수업 중 확인 질문

- 원본 행 수와 정제 행 수가 다른 이유를 설명할 수 있는가?
- 오류가 있는 행을 삭제하지 않고 코드로 남겼는가?
- 중복 기준을 record_id로 잡은 이유를 설명할 수 있는가?
- 저장 파일을 만든 뒤 실제로 존재 여부와 행 수를 확인했는가?
- 같은 노트북을 다시 실행해도 같은 결과가 나오는가?

이 질문에 답할 수 있으면 학생은 단순 크롤링을 넘어 운영 가능한 자동화의 마무리 과정을 이해한 것이다.


## 검증 코드 읽는 순서

학생이 코드를 읽을 때는 함수 정의부터 외우려 하지 말고 입력 데이터의 흐름을 따라가게 한다. 첫째, raw_product_feed.csv에서 어떤 컬럼이 들어오는지 확인한다. 둘째, category_rules.json이 어떤 기준을 제공하는지 확인한다. 셋째, validate_feed_row가 원본 행과 규칙을 함께 받아 어떤 오류 코드를 만드는지 본다. 넷째, checked와 clean_rows가 서로 다른 목적을 가진 리스트라는 점을 구분한다.

checked는 검증 흔적을 남기기 위한 목록이다. 오류가 있어도 행을 보존하고 errors에 이유를 기록한다. clean_rows는 저장과 조회에 쓸 목록이다. 여기에는 중복이 제거되고, 가격과 재고가 숫자로 바뀐 행만 들어간다. 두 리스트의 목적을 구분하지 못하면 학생은 오류 행을 삭제하거나, 반대로 저장하면 안 되는 행까지 CSV에 넣게 된다.

## 오류 코드를 설계하는 방법

오류 코드는 짧지만 구체적이어야 한다. missing:title은 제목이 비었다는 뜻이고, invalid:category는 허용 목록에 없는 category가 들어왔다는 뜻이다. duplicate:record_id는 데이터 형식 문제가 아니라 저장 기준 문제다. 수업에서는 이 세 종류를 분리해서 설명한다.

- 누락 오류: 필수 값이 비어 있어서 운영자가 읽을 수 없다.
- 형식 오류: 값은 있지만 숫자, 상태, 범위 규칙을 통과하지 못한다.
- 중복 오류: 값 자체는 정상이어도 같은 항목을 두 번 저장할 위험이 있다.

이 구분을 해두면 피드백도 구체적으로 줄 수 있다. “틀렸다”가 아니라 “가격 문자열이 숫자로 변환되지 않았다”, “중복 항목을 저장 직전에 걸러야 한다”처럼 수정 방향이 분명해진다.

## 저장 산출물 검토 방식

레슨 끝에서는 학생이 만든 CSV, JSON, SQLite 파일을 모두 열어볼 필요는 없다. 수업 시간에는 대표 확인만 한다. CSV는 첫 행과 행 수, JSON은 raw_count와 clean_count, SQLite는 select count(*) 결과를 확인한다. 세 값이 서로 연결되면 학생이 저장 과정을 이해했다고 볼 수 있다.

최종 미션을 제출할 때는 파일명만 받지 말고 요약 문장도 함께 받는다. 요약 문장에는 원본 행 수, 정제 행 수, 제외 이유, 다음 실행 시 주의점이 들어가야 한다. 이 네 가지가 들어가면 실제 운영자가 자동화 결과를 다시 확인할 수 있다.


## 실제 운영으로 연결하기

실제 운영 화면에서는 수집 데이터가 한 번 저장되면 여러 기능에서 재사용된다. 예를 들어 공지 목록에서 잘못된 제목을 저장하면 메시지 발송 화면에도 같은 제목이 보이고, 가격이나 상태값이 잘못 저장되면 대시보드 집계도 틀어진다. 그래서 저장 전 검증은 부가 기능이 아니라 자동화의 마지막 안전장치다.

학생에게는 “코드가 돌아갔다”와 “운영에 넣어도 된다”를 분리해서 생각하게 한다. 코드가 돌아가도 필수 컬럼이 비어 있거나, 중복 record_id가 섞였거나, 숫자로 비교해야 할 값이 문자열로 남아 있으면 아직 운영 가능한 결과가 아니다. 이번 레슨의 모든 출력은 이 차이를 확인하기 위한 장치다.

수업 중에는 저장 파일을 만든 뒤 반드시 한 번 더 읽어보게 한다. CSV는 DictReader로 다시 읽어 행 수를 확인하고, JSON은 json.loads로 다시 읽어 key가 남아 있는지 확인한다. SQLite는 select count(*)처럼 가장 단순한 조회부터 실행한다. 저장 후 재검증까지 해보면 학생은 자동화 결과를 더 신중하게 다루게 된다.
