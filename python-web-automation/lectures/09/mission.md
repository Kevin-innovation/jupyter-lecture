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

# 레슨 09 — 실습 문제

속도 제한, 재시도, 로그 레슨의 학생용 문제 노트북이다. 강의 노트북을 먼저 실행한 뒤 빈칸을 직접 채운다.

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
    DATA_BASE = 'https://raw.githubusercontent.com/Kevin-innovation/jupyter-lecture/main/python-web-automation/lectures/09/data'
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

class PlannedFetcher:
    def __init__(self, response_rows):
        self.plan = {}
        self.cursor = {}
        for row in response_rows:
            self.plan.setdefault(row['endpoint_id'], []).append(row)
        for endpoint_id in self.plan:
            self.plan[endpoint_id].sort(key=lambda row: int(row['attempt']))
            self.cursor[endpoint_id] = 0
    def get(self, endpoint_id):
        rows = self.plan.get(endpoint_id, [])
        if not rows:
            return {'endpoint_id': endpoint_id, 'attempt': 1, 'status_code': 404, 'body': 'missing-plan'}
        idx = min(self.cursor[endpoint_id], len(rows) - 1)
        self.cursor[endpoint_id] += 1
        row = dict(rows[idx])
        row['status_code'] = int(row['status_code'])
        row['attempt'] = int(row['attempt'])
        return row

def should_retry(status_code):
    return int(status_code) in {408, 429, 500, 502, 503, 504}

def backoff_delay(attempt, base=1, cap=8):
    return min(cap, base * (2 ** max(0, attempt - 1)))

def parse_disallow(text):
    blocked = []
    for line in text.splitlines():
        line = line.strip()
        if line.lower().startswith('disallow:'):
            path = line.split(':', 1)[1].strip()
            if path:
                blocked.append(path)
    return blocked
~~~

---

## 문제 1 — 수집 대상 목록 읽기

지시된 값을 코드로 추출하거나 상태를 변경한다. 실행 결과는 한 줄 출력, 리스트, 딕셔너리, 저장 파일 생성 중 하나다.

**기대 결과 형태**: 요구한 값이 출력되거나 지정한 파일이 생성된다.

**빈칸 힌트**: 대상 CSV 파일과 변수명을 채운다. 완성 코드를 그대로 따라 쓰지 말고 `____` 부분만 스스로 판단한다.

~~~python
targets = load_csv('____')
print(len(____))
~~~

---

## 문제 2 — 응답 계획 읽기

지시된 값을 코드로 추출하거나 상태를 변경한다. 실행 결과는 한 줄 출력, 리스트, 딕셔너리, 저장 파일 생성 중 하나다.

**기대 결과 형태**: 요구한 값이 출력되거나 지정한 파일이 생성된다.

**빈칸 힌트**: 계획 CSV와 endpoint 식별 컬럼을 확인한다. 완성 코드를 그대로 따라 쓰지 말고 `____` 부분만 스스로 판단한다.

~~~python
plan = load_csv('____')
print(plan[0]['____'])
~~~

---

## 문제 3 — robots 차단 경로 파싱

지시된 값을 코드로 추출하거나 상태를 변경한다. 실행 결과는 한 줄 출력, 리스트, 딕셔너리, 저장 파일 생성 중 하나다.

**기대 결과 형태**: 요구한 값이 출력되거나 지정한 파일이 생성된다.

**빈칸 힌트**: robots 샘플 파일 이름과 결과 변수를 채운다. 완성 코드를 그대로 따라 쓰지 말고 `____` 부분만 스스로 판단한다.

~~~python
blocked = parse_disallow(load_text('____'))
print(____)
~~~

---

## 문제 4 — 허용 대상만 필터링

지시된 값을 코드로 추출하거나 상태를 변경한다. 실행 결과는 한 줄 출력, 리스트, 딕셔너리, 저장 파일 생성 중 하나다.

**기대 결과 형태**: 요구한 값이 출력되거나 지정한 파일이 생성된다.

**빈칸 힌트**: 차단 경로 목록과 허용 결과 변수를 넣는다. 완성 코드를 그대로 따라 쓰지 말고 `____` 부분만 스스로 판단한다.

~~~python
allowed = []
for row in targets:
    path = urlparse(row['url']).path
    if not any(path.startswith(rule) for rule in ____):
        allowed.append(row)
print([row['endpoint_id'] for row in ____])
~~~

---

## 문제 5 — PlannedFetcher 첫 응답 확인

지시된 값을 코드로 추출하거나 상태를 변경한다. 실행 결과는 한 줄 출력, 리스트, 딕셔너리, 저장 파일 생성 중 하나다.

**기대 결과 형태**: 요구한 값이 출력되거나 지정한 파일이 생성된다.

**빈칸 힌트**: 응답 계획과 상태 코드 키를 넣는다. 완성 코드를 그대로 따라 쓰지 말고 `____` 부분만 스스로 판단한다.

~~~python
fetcher = PlannedFetcher(____)
response = fetcher.get('notice')
print(response['____'])
~~~

---

## 문제 6 — 재시도 상태 코드 판정

지시된 값을 코드로 추출하거나 상태를 변경한다. 실행 결과는 한 줄 출력, 리스트, 딕셔너리, 저장 파일 생성 중 하나다.

**기대 결과 형태**: 요구한 값이 출력되거나 지정한 파일이 생성된다.

**빈칸 힌트**: 재시도 판정 함수명을 채운다. 완성 코드를 그대로 따라 쓰지 말고 `____` 부분만 스스로 판단한다.

~~~python
for status in [200, 404, 429, 503]:
    print(status, ____(status))
~~~

---

## 문제 7 — backoff 지연 목록 만들기

지시된 값을 코드로 추출하거나 상태를 변경한다. 실행 결과는 한 줄 출력, 리스트, 딕셔너리, 저장 파일 생성 중 하나다.

**기대 결과 형태**: 요구한 값이 출력되거나 지정한 파일이 생성된다.

**빈칸 힌트**: 1~5회 시도 지연을 만들 범위를 채운다. 완성 코드를 그대로 따라 쓰지 말고 `____` 부분만 스스로 판단한다.

~~~python
delays = [backoff_delay(i) for i in range(1, ____)]
print(delays)
~~~

---

## 문제 8 — 우선순위 순서 정렬

지시된 값을 코드로 추출하거나 상태를 변경한다. 실행 결과는 한 줄 출력, 리스트, 딕셔너리, 저장 파일 생성 중 하나다.

**기대 결과 형태**: 요구한 값이 출력되거나 지정한 파일이 생성된다.

**빈칸 힌트**: 우선순위 컬럼명을 넣는다. 완성 코드를 그대로 따라 쓰지 말고 `____` 부분만 스스로 판단한다.

~~~python
ordered = sorted(allowed, key=lambda row: int(row['____']))
print([row['endpoint_id'] for row in ordered])
~~~

---

## 문제 9 — 요청 간격 숫자 변환

지시된 값을 코드로 추출하거나 상태를 변경한다. 실행 결과는 한 줄 출력, 리스트, 딕셔너리, 저장 파일 생성 중 하나다.

**기대 결과 형태**: 요구한 값이 출력되거나 지정한 파일이 생성된다.

**빈칸 힌트**: 최소 간격 컬럼명을 채운다. 완성 코드를 그대로 따라 쓰지 말고 `____` 부분만 스스로 판단한다.

~~~python
intervals = {row['endpoint_id']: float(row['____']) for row in allowed}
print(intervals)
~~~

---

## 문제 10 — 단일 endpoint 재시도 루프

지시된 값을 코드로 추출하거나 상태를 변경한다. 실행 결과는 한 줄 출력, 리스트, 딕셔너리, 저장 파일 생성 중 하나다.

**기대 결과 형태**: 요구한 값이 출력되거나 지정한 파일이 생성된다.

**빈칸 힌트**: 성공 또는 중단 조건에서 반복을 멈춘다. 완성 코드를 그대로 따라 쓰지 말고 `____` 부분만 스스로 판단한다.

~~~python
fetcher = PlannedFetcher(plan)
logs = []
for attempt_no in range(1, 4):
    response = fetcher.get('status')
    logs.append({'endpoint_id': 'status', 'attempt_no': attempt_no, 'status_code': response['status_code']})
    if response['status_code'] == 200 or not should_retry(response['status_code']):
        ____
print(logs)
~~~

---

## 문제 11 — fetch_with_retry 함수 작성

지시된 값을 코드로 추출하거나 상태를 변경한다. 실행 결과는 한 줄 출력, 리스트, 딕셔너리, 저장 파일 생성 중 하나다.

**기대 결과 형태**: 요구한 값이 출력되거나 지정한 파일이 생성된다.

**빈칸 힌트**: 마지막 응답과 로그를 반환한다. 완성 코드를 그대로 따라 쓰지 말고 `____` 부분만 스스로 판단한다.

~~~python
def fetch_with_retry(fetcher, endpoint_id, max_attempts=3):
    logs = []
    for attempt_no in range(1, max_attempts + 1):
        response = fetcher.get(endpoint_id)
        logs.append({'endpoint_id': endpoint_id, 'attempt_no': attempt_no, 'status_code': response['status_code'], 'retry': should_retry(response['status_code'])})
        if response['status_code'] == 200 or not should_retry(response['status_code']):
            return response, logs
    return ____, ____
response, logs = fetch_with_retry(PlannedFetcher(plan), 'notice')
print(response['status_code'], len(logs))
~~~

---

## 문제 12 — 전체 허용 endpoint 실행

지시된 값을 코드로 추출하거나 상태를 변경한다. 실행 결과는 한 줄 출력, 리스트, 딕셔너리, 저장 파일 생성 중 하나다.

**기대 결과 형태**: 요구한 값이 출력되거나 지정한 파일이 생성된다.

**빈칸 힌트**: endpoint 식별 컬럼을 넣는다. 완성 코드를 그대로 따라 쓰지 말고 `____` 부분만 스스로 판단한다.

~~~python
fetcher = PlannedFetcher(plan)
all_logs = []
final = []
for row in ordered:
    response, logs = fetch_with_retry(fetcher, row['____'])
    all_logs.extend(logs)
    final.append({'endpoint_id': row['endpoint_id'], 'status_code': response['status_code']})
print(final)
~~~

---

## 문제 13 — 시도 로그 CSV 저장

지시된 값을 코드로 추출하거나 상태를 변경한다. 실행 결과는 한 줄 출력, 리스트, 딕셔너리, 저장 파일 생성 중 하나다.

**기대 결과 형태**: 요구한 값이 출력되거나 지정한 파일이 생성된다.

**빈칸 힌트**: 전체 시도 로그 목록을 넣는다. 완성 코드를 그대로 따라 쓰지 말고 `____` 부분만 스스로 판단한다.

~~~python
write_csv('lesson09_attempt_log.csv', ____, fieldnames=['endpoint_id', 'attempt_no', 'status_code', 'retry'])
print(Path('lesson09_attempt_log.csv').exists())
~~~

---

## 문제 14 — 최종 요약 JSON 저장

지시된 값을 코드로 추출하거나 상태를 변경한다. 실행 결과는 한 줄 출력, 리스트, 딕셔너리, 저장 파일 생성 중 하나다.

**기대 결과 형태**: 요구한 값이 출력되거나 지정한 파일이 생성된다.

**빈칸 힌트**: 성공 상태 코드를 채운다. 완성 코드를 그대로 따라 쓰지 말고 `____` 부분만 스스로 판단한다.

~~~python
summary = {'target_count': len(allowed), 'attempt_count': len(all_logs), 'success_count': sum(1 for row in final if row['status_code'] == ____)}
Path('lesson09_retry_summary.json').write_text(json.dumps(summary, ensure_ascii=False, indent=2), encoding='utf-8')
print(summary)
~~~

---

## 문제 15 — 운영 메모 문장 만들기

지시된 값을 코드로 추출하거나 상태를 변경한다. 실행 결과는 한 줄 출력, 리스트, 딕셔너리, 저장 파일 생성 중 하나다.

**기대 결과 형태**: 요구한 값이 출력되거나 지정한 파일이 생성된다.

**빈칸 힌트**: 템플릿 파일과 출력할 변수를 채운다. 완성 코드를 그대로 따라 쓰지 말고 `____` 부분만 스스로 판단한다.

~~~python
templates = load_json('____')
message = templates['summary_template'].format(success=summary['success_count'], total=summary['target_count'], attempts=summary['attempt_count'])
print(____)
~~~

### 보강 설명 1

문제가 막히면 HTML 태그, CSV 헤더, JSON 키를 먼저 소리 내어 읽는다. 함수명을 떠올리기 전에 입력 데이터의 구조를 확인하면 빈칸을 더 안정적으로 채울 수 있다.

### 보강 설명 2

문제가 막히면 HTML 태그, CSV 헤더, JSON 키를 먼저 소리 내어 읽는다. 함수명을 떠올리기 전에 입력 데이터의 구조를 확인하면 빈칸을 더 안정적으로 채울 수 있다.

### 보강 설명 3

문제가 막히면 HTML 태그, CSV 헤더, JSON 키를 먼저 소리 내어 읽는다. 함수명을 떠올리기 전에 입력 데이터의 구조를 확인하면 빈칸을 더 안정적으로 채울 수 있다.
