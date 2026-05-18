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

[![Open in Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/Kevin-innovation/jupyter-lecture/blob/main/python-web-automation/lectures/08/%EB%A0%88%EC%8A%A8%2008%20%E2%80%94%20%EC%88%98%EC%A7%91%20%EB%8D%B0%EC%9D%B4%ED%84%B0%20%EA%B2%80%EC%A6%9D%EA%B3%BC%20%EC%A0%80%EC%9E%A5.ipynb)

> 선생님용 강의 노트북이다. 코랩에서 확인하려면 우측 상단 또는 아래의 **Open in Colab** 버튼을 클릭한다.

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

원본 CSV를 먼저 읽어야 이후 모든 검증의 기준 행 수를 잡을 수 있다. 행 수는 저장 전후 비교의 기준값이므로 별도 변수 rows에 남긴다. 파일명을 추측하지 않고 fixture 이름을 정확히 사용하는 것이 핵심이다.

---

## 문제 2 정답 — 검증 규칙 JSON 읽기

~~~python
rules = load_json('category_rules.json')
print(rules['required_fields'])
~~~

### 왜 이 코드가 정답인지

검증 규칙은 코드 안에 흩뿌리지 않고 JSON에서 읽는다. required_fields를 출력하면 어떤 컬럼이 반드시 필요한지 학생이 먼저 확인할 수 있고, 이후 validate 함수의 기준과 연결된다.

---

## 문제 3 정답 — 가격 문자열을 숫자로 변환

~~~python
price = parse_price(rows[0]['price_text'])
print(price)
~~~

### 왜 이 코드가 정답인지

가격 문자열은 쉼표와 원 표시가 섞여 있으므로 숫자만 남기는 변환이 필요하다. price_text를 직접 int로 바꾸면 실패하므로 parse_price를 통해 저장 가능한 정수로 정규화한다.

---

## 문제 4 정답 — 재고 값을 정수로 변환

~~~python
stock = parse_stock(rows[0]['stock'])
print(stock)
~~~

### 왜 이 코드가 정답인지

재고는 문자열로 읽히지만 저장과 비교에는 정수가 필요하다. parse_stock은 정상 숫자를 int로 바꾸고 실패 입력은 None으로 남겨 검증 단계에서 오류로 분류할 수 있게 한다.

---

## 문제 5 정답 — 첫 행 오류 코드 확인

~~~python
errors = validate_feed_row(rows[0], rules)
print(errors)
~~~

### 왜 이 코드가 정답인지

한 행을 규칙과 함께 검증해야 누락, 범위, 허용값 오류를 모두 잡을 수 있다. rules를 전달하지 않으면 어떤 category와 status가 허용되는지 알 수 없다.

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

전체 행에 같은 검증 함수를 적용하고 errors와 is_valid를 함께 붙이면 원본과 판정 결과를 한 행에서 추적할 수 있다. 오류를 즉시 삭제하지 않는 점이 운영 자동화에서 중요하다.

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

중복은 저장 전에 별도로 세야 한다. record_id를 seen set에 모아두면 두 번째 등장한 키만 duplicates에 남길 수 있고, 중복 리포트의 근거가 생긴다.

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

정제 리스트는 중복을 건너뛰고 검증을 통과한 행만 저장한다. 이때 가격과 재고를 숫자로 바꾸고 필요한 필드만 남기므로 이후 CSV, JSON, SQLite 저장에 바로 사용할 수 있다.

---

## 문제 9 정답 — 카테고리별 개수 요약

~~~python
summary = {}
for row in clean_rows:
    summary[row['category']] = summary.get(row['category'], 0) + 1
print(summary)
~~~

### 왜 이 코드가 정답인지

카테고리별 개수는 정제 데이터가 실제로 어떤 구성을 갖는지 보여주는 첫 번째 운영 요약이다. 같은 category를 key로 묶고 count를 누적하면 저장 결과를 빠르게 검토할 수 있다.

---

## 문제 10 정답 — 정제 CSV 저장

~~~python
write_csv('lesson08_clean_feed.csv', clean_rows)
print(Path('lesson08_clean_feed.csv').exists())
~~~

### 왜 이 코드가 정답인지

CSV 저장은 운영자가 표로 확인하기 위한 산출물이다. 저장 후 파일 존재 여부를 바로 출력하면 코드가 실행만 된 것이 아니라 실제 산출물을 만들었는지 확인할 수 있다.

---

## 문제 11 정답 — 품질 리포트 JSON 저장

~~~python
report = {'raw_count': len(rows), 'clean_count': len(clean_rows), 'duplicate_count': len(duplicates)}
Path('lesson08_quality_report.json').write_text(json.dumps(report, ensure_ascii=False, indent=2), encoding='utf-8')
print(report)
~~~

### 왜 이 코드가 정답인지

품질 리포트에는 원본, 정제, 중복 수가 함께 있어야 한다. 이 값들이 맞아야 왜 일부 행이 제외되었는지 설명할 수 있고, 다음 수집 기준을 조정할 근거가 생긴다.

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

SQLite는 정제 행을 다시 질의할 수 있는 구조로 바꾼다. 테이블을 만들고 clean_rows를 삽입한 뒤 count를 조회하면 저장된 행 수가 기대와 맞는지 확인할 수 있다.

---

## 문제 13 정답 — 가격 높은 재고 상품 조회

~~~python
conn = sqlite3.connect('lesson08_feed.db')
rows_sql = conn.execute('select title, price from feed where stock > 0 and price >= ? order by price desc', (50000,)).fetchall()
print(rows_sql[:5])
conn.close()
~~~

### 왜 이 코드가 정답인지

가격과 재고 조건을 SQL 파라미터로 조회하면 저장 후 활용 흐름을 보여줄 수 있다. 문자열 조합 대신 파라미터 바인딩을 사용하면 조건값을 바꿔도 쿼리 구조가 안정적이다.

---

## 문제 14 정답 — HTML 표에서 행 개수 읽기

~~~python
soup = BeautifulSoup(load_text('validation_panel.html'), 'html.parser')
items = soup.select('tbody tr')
print(len(items))
~~~

### 왜 이 코드가 정답인지

HTML 표 fixture는 웹 화면에서 같은 데이터를 확인하는 상황을 연습한다. tbody tr를 선택하면 헤더가 아니라 실제 데이터 행만 세므로 CSV 행 수와 비교할 수 있다.

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

파이프라인 함수는 읽기, 검증, 중복 판정, 정제를 한 번에 재실행 가능하게 묶는다. checked와 clean을 같이 반환해야 오류 리포트와 저장용 데이터를 동시에 관리할 수 있다.

## 교사용 검토 메모

이번 정답지는 값 하나를 맞히는 용도가 아니라 데이터 검증 파이프라인을 확인하기 위한 기준이다. 학생 답안이 모범 코드와 조금 달라도 다음 조건을 만족하면 인정할 수 있다.

- 필수 컬럼 검증이 누락되지 않았다.
- 가격과 재고가 저장 전에 숫자로 정규화되었다.
- 오류 행을 조용히 삭제하지 않고 errors 또는 별도 리포트에 남겼다.
- 중복 record_id를 정제 저장 대상에서 제외했다.
- CSV 또는 JSON 산출물을 만들고 존재 여부를 확인했다.
- 최종 함수가 다시 실행되어도 같은 행 수를 만든다.

반대로 출력값이 우연히 맞아도 검증 기준이 코드에 드러나지 않거나, 오류 행을 설명 없이 버렸다면 다시 보완하게 한다. 운영 자동화에서는 성공 결과보다 실패를 추적하는 구조가 더 중요하다.


## 문제별 채점 포인트

### 문제 1 채점 포인트

학생이 원본 피드 파일명을 정확히 입력했는지 확인한다. 행 수만 맞아도 변수에 rows를 남기지 않으면 이후 문제에서 다시 읽어야 하므로 감점할 수 있다. 원본 행 수는 정제 행 수와 비교하는 기준이다.

### 문제 2 채점 포인트

category_rules.json에서 required_fields를 직접 읽는지 확인한다. 필수 컬럼 목록을 코드에 직접 하드코딩하면 현재 문제는 맞아도 규칙 파일이 바뀔 때 자동화가 따라가지 못한다.

### 문제 3 채점 포인트

가격 문자열을 숫자로 변환하는 이유를 설명할 수 있어야 한다. 쉼표나 원 표시가 남아 있으면 SQLite 조회와 가격 비교가 불가능하므로 parse_price 사용 여부를 본다.

### 문제 4 채점 포인트

재고는 음수와 잘못된 문자열을 잡아야 한다. parse_stock이 실패 값을 None으로 돌려주고, validate 단계에서 invalid:stock으로 분류되는 흐름을 설명하면 충분하다.

### 문제 5 채점 포인트

첫 행 검증은 함수 호출 형태를 보는 문제다. row만 넣거나 rules만 넣으면 검증 기준이 완성되지 않는다. errors가 빈 리스트일 수 있다는 것도 정상 결과로 인정한다.

### 문제 6 채점 포인트

checked 리스트에 원본 필드와 판정 필드가 함께 들어가는지 확인한다. errors를 문자열로 합치는 이유는 CSV나 표에서 사람이 읽기 쉽게 만들기 위한 것이다. is_valid는 not errors로 계산해야 한다.

### 문제 7 채점 포인트

중복 record_id를 발견한 뒤 즉시 삭제하지 않고 duplicates에 기록하는지 본다. 중복 목록이 있어야 품질 리포트에서 왜 정제 행 수가 줄었는지 설명할 수 있다.

### 문제 8 채점 포인트

정제 리스트에는 유효 행만 들어가야 하며 중복은 한 번만 저장되어야 한다. title은 strip으로 공백을 정리하고 price, stock은 숫자로 저장하는지 확인한다.

### 문제 9 채점 포인트

카테고리 요약은 저장 결과의 균형을 확인하는 간단한 집계다. summary.get(row['category'], 0) + 1 흐름을 이해하면 다른 상태값 요약으로도 확장할 수 있다.

### 문제 10 채점 포인트

CSV 저장 후 파일 존재를 확인하는 습관을 본다. write_csv만 호출하고 끝내면 실제 저장 성공 여부가 보이지 않는다. 산출물 파일명은 문제에서 지정한 이름을 유지해야 한다.

### 문제 11 채점 포인트

JSON 리포트는 운영 요약이다. raw_count, clean_count, duplicate_count가 모두 있어야 원본과 결과 차이를 설명할 수 있다. ensure_ascii=False를 사용하면 한글 리포트가 읽기 좋다.

### 문제 12 채점 포인트

SQLite 테이블 생성은 저장 후 조회 가능성을 보여준다. drop table로 재실행성을 확보하고, executemany에 clean_rows를 넣어야 같은 노트북을 다시 실행해도 깨지지 않는다.

### 문제 13 채점 포인트

SQL 조건 조회는 저장 데이터가 실제 질문에 답할 수 있는지 확인한다. stock > 0과 price >= ? 조건을 함께 사용하면 판매 가능한 고가 항목을 찾는 운영 예시가 된다.

### 문제 14 채점 포인트

HTML 표에서는 tbody tr를 선택해야 데이터 행만 센다. table tr 전체를 선택하면 헤더가 있는 HTML에서는 행 수가 달라질 수 있음을 짚어준다.

### 문제 15 채점 포인트

최종 함수는 읽기, 검증, 중복 처리, 정제를 재사용 가능한 단위로 묶어야 한다. checked와 clean을 모두 반환해야 오류 추적과 저장이 동시에 가능하다. 함수 안에서 파일명을 다시 읽기 때문에 다른 노트북에서도 독립적으로 실행된다.

## 오답 유형과 피드백 문장

- CSV를 바로 저장한 경우: “저장 전에 검증 결과를 남겨야 다음 실행에서 제외 이유를 설명할 수 있다.”
- 오류 행을 삭제한 경우: “오류 행도 리포트에는 남겨야 원본 품질을 개선할 수 있다.”
- 중복을 세지 않은 경우: “정제 행 수가 줄어든 이유를 duplicate_count로 설명해야 한다.”
- 문자열 가격을 그대로 저장한 경우: “가격 비교와 SQL 조회를 하려면 숫자로 정규화해야 한다.”
- 파일 존재 확인이 없는 경우: “자동화는 산출물이 실제로 만들어졌는지 마지막에 확인해야 한다.”

## 수업 후 점검 기준

이번 레슨을 마친 학생은 “수집 데이터는 바로 저장하지 않는다”는 원칙을 말할 수 있어야 한다. 또한 원본 행, 검증 행, 정제 행, 저장 파일이 서로 다른 역할을 가진다는 점을 구분해야 한다. 이 구분이 되면 이후 9강의 알림/리포트 자동화에서도 잘못된 데이터를 보내는 위험을 줄일 수 있다.


## 단계별 해설 보충

### 원본과 규칙을 분리해서 읽는 이유

원본 데이터와 검증 규칙을 같은 파일에 섞어두면 자동화가 커질수록 관리가 어려워진다. 이번 레슨에서는 raw_product_feed.csv가 바뀌어도 category_rules.json의 기준은 그대로 유지될 수 있고, 반대로 운영 기준이 바뀌면 JSON만 바꿔 다시 실행할 수 있다. 학생이 이 구조를 이해하면 이후 수업에서 요청 간격, 허용 도메인, 저장 경로 같은 설정도 별도 파일로 분리할 수 있다.

### 정규화 함수의 역할

parse_price와 parse_stock은 단순한 편의 함수가 아니다. 저장 전에 데이터 타입을 확정하는 단계다. 가격이 문자열로 남아 있으면 “5만원 이상” 같은 조건을 안정적으로 처리할 수 없고, 재고가 문자열로 남아 있으면 음수 검증도 불안정해진다. 함수가 작아 보여도 운영 자동화에서는 입력을 저장 가능한 형태로 바꾸는 경계 역할을 한다.

### validate_feed_row 설계 의도

검증 함수는 한 행을 받아 오류 목록을 돌려준다. 참/거짓 하나만 돌려주지 않는 이유는 학생과 운영자가 실패 원인을 알아야 하기 때문이다. 예를 들어 제목도 비었고 category도 잘못된 행은 두 오류를 모두 남겨야 한다. 하나만 남기면 첫 번째 오류를 고친 뒤 두 번째 오류를 다시 발견하게 되어 피드백 시간이 길어진다.

### checked와 clean_rows를 나누는 이유

checked에는 원본과 오류 정보가 함께 들어간다. 이 목록은 피드백과 품질 리포트에 쓰인다. clean_rows에는 운영 저장에 필요한 필드만 들어간다. 이 목록은 CSV, JSON, SQLite에 쓰인다. 두 목록을 하나로 합치면 오류 행을 저장하거나, 반대로 오류 리포트를 잃어버리는 문제가 생긴다. 그래서 수업에서는 두 변수 이름을 계속 비교해서 읽게 한다.

### 중복 처리 위치

중복은 validate_feed_row 안에 넣을 수도 있지만, 이 레슨에서는 별도 반복문과 최종 함수에서 처리한다. 이유는 중복 기준이 필드 값의 형식 오류와 성격이 다르기 때문이다. price나 status는 한 행만 봐도 검증할 수 있지만, duplicate는 앞에서 같은 record_id가 나왔는지 기억해야 판단할 수 있다. 이 차이를 이해하면 학생이 상태를 기억하는 set의 필요성을 자연스럽게 받아들인다.

### 저장 형식별 피드백

CSV를 저장한 학생에게는 헤더와 행 수를 먼저 보게 한다. JSON을 저장한 학생에게는 key 이름과 값의 의미를 설명하게 한다. SQLite를 저장한 학생에게는 직접 쿼리를 하나 더 만들어보게 한다. 같은 데이터라도 저장 형식이 달라지면 검토 방법도 달라진다는 점이 이번 레슨의 중요한 포인트다.

### 최종 함수의 재실행성

build_clean_feed는 셀을 다시 실행해도 같은 결과를 만들어야 한다. 함수 내부에서 rows, rules, checked, seen, clean을 새로 만들기 때문에 이전 실행 상태에 덜 의존한다. 학생이 전역 변수에만 기대면 노트북 실행 순서가 바뀌었을 때 결과가 달라질 수 있다. 함수로 묶는 연습은 코랩 수업에서도 실무적인 안정성을 높인다.

## 예상 출력 확인

원본 피드는 오류와 중복을 포함한 40행이다. 유효하면서 중복이 아닌 정제 행은 그보다 적다. 학생 답안에서 정확한 숫자가 다르다면 먼저 validate_feed_row 기준을 확인한다. 가격문의, 빈 제목, private category, 음수 stock, hidden status, 중복 P006이 제대로 제외되었는지 보면 대부분의 오류를 찾을 수 있다.

HTML 표 행 수는 CSV 전체 행 수와 다를 수 있다. validation_panel.html은 일부 행만 웹 패널에 표시하는 fixture이므로, 학생이 CSV 행 수와 HTML 행 수를 무조건 같다고 가정하면 안 된다. 이 차이는 실제 운영 화면과 원본 수집 데이터가 항상 1:1로 대응하지 않는다는 설명으로 연결한다.

## 교사용 빠른 판정표

| 항목 | 통과 기준 | 다시 볼 신호 |
| --- | --- | --- |
| 원본 읽기 | rows가 CSV 전체 행을 담는다 | 파일명을 잘못 입력하거나 행 수가 0이다 |
| 규칙 읽기 | required_fields와 valid 목록을 JSON에서 읽는다 | 기준을 코드에 직접 적었다 |
| 정규화 | 가격과 재고가 숫자로 바뀐다 | 원 표시, 쉼표가 남아 있다 |
| 검증 | errors에 구체적 코드가 남는다 | True/False만 남기고 이유가 없다 |
| 중복 | duplicate 목록이나 코드가 있다 | 중복 P006이 그대로 저장된다 |
| 저장 | CSV/JSON/SQLite 중 산출물이 생긴다 | 저장 호출 후 확인 출력이 없다 |
| 요약 | raw, clean, duplicate 수를 설명한다 | 행 수 차이를 말하지 못한다 |

## 수업 마무리 피드백 예시

좋은 답안은 코드가 길지 않아도 기준이 분명하다. “필수 컬럼을 확인하고, 가격과 재고를 숫자로 바꾼 뒤, 오류 행과 중복 행을 저장 대상에서 제외했다”라고 설명할 수 있으면 핵심을 이해한 것이다. 보완이 필요한 답안은 대체로 저장 파일은 만들었지만 왜 그 행들이 저장되었는지 설명하지 못한다. 이때는 정답 코드를 보여주기보다 errors와 duplicate_count를 직접 출력하게 하는 편이 효과적이다.


## 재실행 검증 기준

교사용 노트북은 한 번 실행된 상태에서만 맞으면 충분하지 않다. 커널을 재시작한 뒤 위에서 아래로 다시 실행해도 같은 CSV, JSON, SQLite 파일이 만들어져야 한다. 특히 SQLite 셀은 drop table을 먼저 실행하므로 같은 셀을 반복 실행해도 테이블 중복 오류가 나지 않는다. CSV와 JSON은 같은 파일명을 덮어쓰는 구조라서 수업 중 여러 번 실행해도 산출물 이름이 늘어나지 않는다.

학생 답안에서 이전 셀의 우연한 상태에 기대는 코드가 보이면 재실행 검증으로 바로 드러난다. 예를 들어 rows나 rules를 만들지 않고 중간 문제부터 실행하면 실패하는 것은 자연스럽지만, 전체 실행에서도 실패한다면 변수 생성 순서가 잘못된 것이다. 최종 피드백은 “한 셀만 맞히기”보다 “전체 노트북을 다시 실행해도 같은 결과가 나오는지”를 기준으로 준다.
