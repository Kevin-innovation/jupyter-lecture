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

# 레슨 09 — 실습 문제 정답지

> 🔒 교사·관리자 전용. 학생에게 배포 금지.

속도 제한, 재시도, 로그 실습 문제의 모범 답안이다. 출력값만 보지 말고 검증 기준, 저장 구조, 로그를 함께 확인한다.

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

## 문제 1 정답 — 수집 대상 목록 읽기

~~~python
targets = load_csv('targets.csv')
print(len(targets))
~~~

### 왜 이 코드가 정답인지

이 답안은 문제에서 요구한 입력 구조를 먼저 읽고, 필요한 컬럼이나 selector만 선택한 뒤 결과를 명시적인 변수에 저장한다. `수집 대상 목록 읽기` 단계에서 중요한 점은 출력 하나를 맞히는 것이 아니라 이후 문제에서 재사용할 수 있는 안정적인 중간 결과를 남기는 것이다. 빈칸에는 파일명, 컬럼명, 키 이름, 반복 제어 중 핵심 위치만 남겼기 때문에 학생이 데이터 구조를 이해해야 완성할 수 있다. 자주 보이는 오답은 컬럼명을 추측하거나, 중복과 실패 상태를 처리하지 않고 바로 저장하는 방식이다.

---

## 문제 2 정답 — 응답 계획 읽기

~~~python
plan = load_csv('response_plan.csv')
print(plan[0]['endpoint_id'])
~~~

### 왜 이 코드가 정답인지

이 답안은 문제에서 요구한 입력 구조를 먼저 읽고, 필요한 컬럼이나 selector만 선택한 뒤 결과를 명시적인 변수에 저장한다. `응답 계획 읽기` 단계에서 중요한 점은 출력 하나를 맞히는 것이 아니라 이후 문제에서 재사용할 수 있는 안정적인 중간 결과를 남기는 것이다. 빈칸에는 파일명, 컬럼명, 키 이름, 반복 제어 중 핵심 위치만 남겼기 때문에 학생이 데이터 구조를 이해해야 완성할 수 있다. 자주 보이는 오답은 컬럼명을 추측하거나, 중복과 실패 상태를 처리하지 않고 바로 저장하는 방식이다.

---

## 문제 3 정답 — robots 차단 경로 파싱

~~~python
blocked = parse_disallow(load_text('robots_sample.txt'))
print(blocked)
~~~

### 왜 이 코드가 정답인지

이 답안은 문제에서 요구한 입력 구조를 먼저 읽고, 필요한 컬럼이나 selector만 선택한 뒤 결과를 명시적인 변수에 저장한다. `robots 차단 경로 파싱` 단계에서 중요한 점은 출력 하나를 맞히는 것이 아니라 이후 문제에서 재사용할 수 있는 안정적인 중간 결과를 남기는 것이다. 빈칸에는 파일명, 컬럼명, 키 이름, 반복 제어 중 핵심 위치만 남겼기 때문에 학생이 데이터 구조를 이해해야 완성할 수 있다. 자주 보이는 오답은 컬럼명을 추측하거나, 중복과 실패 상태를 처리하지 않고 바로 저장하는 방식이다.

---

## 문제 4 정답 — 허용 대상만 필터링

~~~python
allowed = []
for row in targets:
    path = urlparse(row['url']).path
    if not any(path.startswith(rule) for rule in blocked):
        allowed.append(row)
print([row['endpoint_id'] for row in allowed])
~~~

### 왜 이 코드가 정답인지

이 답안은 문제에서 요구한 입력 구조를 먼저 읽고, 필요한 컬럼이나 selector만 선택한 뒤 결과를 명시적인 변수에 저장한다. `허용 대상만 필터링` 단계에서 중요한 점은 출력 하나를 맞히는 것이 아니라 이후 문제에서 재사용할 수 있는 안정적인 중간 결과를 남기는 것이다. 빈칸에는 파일명, 컬럼명, 키 이름, 반복 제어 중 핵심 위치만 남겼기 때문에 학생이 데이터 구조를 이해해야 완성할 수 있다. 자주 보이는 오답은 컬럼명을 추측하거나, 중복과 실패 상태를 처리하지 않고 바로 저장하는 방식이다.

---

## 문제 5 정답 — PlannedFetcher 첫 응답 확인

~~~python
fetcher = PlannedFetcher(plan)
response = fetcher.get('notice')
print(response['status_code'])
~~~

### 왜 이 코드가 정답인지

이 답안은 문제에서 요구한 입력 구조를 먼저 읽고, 필요한 컬럼이나 selector만 선택한 뒤 결과를 명시적인 변수에 저장한다. `PlannedFetcher 첫 응답 확인` 단계에서 중요한 점은 출력 하나를 맞히는 것이 아니라 이후 문제에서 재사용할 수 있는 안정적인 중간 결과를 남기는 것이다. 빈칸에는 파일명, 컬럼명, 키 이름, 반복 제어 중 핵심 위치만 남겼기 때문에 학생이 데이터 구조를 이해해야 완성할 수 있다. 자주 보이는 오답은 컬럼명을 추측하거나, 중복과 실패 상태를 처리하지 않고 바로 저장하는 방식이다.

---

## 문제 6 정답 — 재시도 상태 코드 판정

~~~python
for status in [200, 404, 429, 503]:
    print(status, should_retry(status))
~~~

### 왜 이 코드가 정답인지

이 답안은 문제에서 요구한 입력 구조를 먼저 읽고, 필요한 컬럼이나 selector만 선택한 뒤 결과를 명시적인 변수에 저장한다. `재시도 상태 코드 판정` 단계에서 중요한 점은 출력 하나를 맞히는 것이 아니라 이후 문제에서 재사용할 수 있는 안정적인 중간 결과를 남기는 것이다. 빈칸에는 파일명, 컬럼명, 키 이름, 반복 제어 중 핵심 위치만 남겼기 때문에 학생이 데이터 구조를 이해해야 완성할 수 있다. 자주 보이는 오답은 컬럼명을 추측하거나, 중복과 실패 상태를 처리하지 않고 바로 저장하는 방식이다.

---

## 문제 7 정답 — backoff 지연 목록 만들기

~~~python
delays = [backoff_delay(i) for i in range(1, 6)]
print(delays)
~~~

### 왜 이 코드가 정답인지

이 답안은 문제에서 요구한 입력 구조를 먼저 읽고, 필요한 컬럼이나 selector만 선택한 뒤 결과를 명시적인 변수에 저장한다. `backoff 지연 목록 만들기` 단계에서 중요한 점은 출력 하나를 맞히는 것이 아니라 이후 문제에서 재사용할 수 있는 안정적인 중간 결과를 남기는 것이다. 빈칸에는 파일명, 컬럼명, 키 이름, 반복 제어 중 핵심 위치만 남겼기 때문에 학생이 데이터 구조를 이해해야 완성할 수 있다. 자주 보이는 오답은 컬럼명을 추측하거나, 중복과 실패 상태를 처리하지 않고 바로 저장하는 방식이다.

---

## 문제 8 정답 — 우선순위 순서 정렬

~~~python
ordered = sorted(allowed, key=lambda row: int(row['priority']))
print([row['endpoint_id'] for row in ordered])
~~~

### 왜 이 코드가 정답인지

이 답안은 문제에서 요구한 입력 구조를 먼저 읽고, 필요한 컬럼이나 selector만 선택한 뒤 결과를 명시적인 변수에 저장한다. `우선순위 순서 정렬` 단계에서 중요한 점은 출력 하나를 맞히는 것이 아니라 이후 문제에서 재사용할 수 있는 안정적인 중간 결과를 남기는 것이다. 빈칸에는 파일명, 컬럼명, 키 이름, 반복 제어 중 핵심 위치만 남겼기 때문에 학생이 데이터 구조를 이해해야 완성할 수 있다. 자주 보이는 오답은 컬럼명을 추측하거나, 중복과 실패 상태를 처리하지 않고 바로 저장하는 방식이다.

---

## 문제 9 정답 — 요청 간격 숫자 변환

~~~python
intervals = {row['endpoint_id']: float(row['min_interval_seconds']) for row in allowed}
print(intervals)
~~~

### 왜 이 코드가 정답인지

이 답안은 문제에서 요구한 입력 구조를 먼저 읽고, 필요한 컬럼이나 selector만 선택한 뒤 결과를 명시적인 변수에 저장한다. `요청 간격 숫자 변환` 단계에서 중요한 점은 출력 하나를 맞히는 것이 아니라 이후 문제에서 재사용할 수 있는 안정적인 중간 결과를 남기는 것이다. 빈칸에는 파일명, 컬럼명, 키 이름, 반복 제어 중 핵심 위치만 남겼기 때문에 학생이 데이터 구조를 이해해야 완성할 수 있다. 자주 보이는 오답은 컬럼명을 추측하거나, 중복과 실패 상태를 처리하지 않고 바로 저장하는 방식이다.

---

## 문제 10 정답 — 단일 endpoint 재시도 루프

~~~python
fetcher = PlannedFetcher(plan)
logs = []
for attempt_no in range(1, 4):
    response = fetcher.get('status')
    logs.append({'endpoint_id': 'status', 'attempt_no': attempt_no, 'status_code': response['status_code']})
    if response['status_code'] == 200 or not should_retry(response['status_code']):
        break
print(logs)
~~~

### 왜 이 코드가 정답인지

이 답안은 문제에서 요구한 입력 구조를 먼저 읽고, 필요한 컬럼이나 selector만 선택한 뒤 결과를 명시적인 변수에 저장한다. `단일 endpoint 재시도 루프` 단계에서 중요한 점은 출력 하나를 맞히는 것이 아니라 이후 문제에서 재사용할 수 있는 안정적인 중간 결과를 남기는 것이다. 빈칸에는 파일명, 컬럼명, 키 이름, 반복 제어 중 핵심 위치만 남겼기 때문에 학생이 데이터 구조를 이해해야 완성할 수 있다. 자주 보이는 오답은 컬럼명을 추측하거나, 중복과 실패 상태를 처리하지 않고 바로 저장하는 방식이다.

---

## 문제 11 정답 — fetch_with_retry 함수 작성

~~~python
def fetch_with_retry(fetcher, endpoint_id, max_attempts=3):
    logs = []
    for attempt_no in range(1, max_attempts + 1):
        response = fetcher.get(endpoint_id)
        logs.append({'endpoint_id': endpoint_id, 'attempt_no': attempt_no, 'status_code': response['status_code'], 'retry': should_retry(response['status_code'])})
        if response['status_code'] == 200 or not should_retry(response['status_code']):
            return response, logs
    return response, logs
response, logs = fetch_with_retry(PlannedFetcher(plan), 'notice')
print(response['status_code'], len(logs))
~~~

### 왜 이 코드가 정답인지

이 답안은 문제에서 요구한 입력 구조를 먼저 읽고, 필요한 컬럼이나 selector만 선택한 뒤 결과를 명시적인 변수에 저장한다. `fetch_with_retry 함수 작성` 단계에서 중요한 점은 출력 하나를 맞히는 것이 아니라 이후 문제에서 재사용할 수 있는 안정적인 중간 결과를 남기는 것이다. 빈칸에는 파일명, 컬럼명, 키 이름, 반복 제어 중 핵심 위치만 남겼기 때문에 학생이 데이터 구조를 이해해야 완성할 수 있다. 자주 보이는 오답은 컬럼명을 추측하거나, 중복과 실패 상태를 처리하지 않고 바로 저장하는 방식이다.

---

## 문제 12 정답 — 전체 허용 endpoint 실행

~~~python
fetcher = PlannedFetcher(plan)
all_logs = []
final = []
for row in ordered:
    response, logs = fetch_with_retry(fetcher, row['endpoint_id'])
    all_logs.extend(logs)
    final.append({'endpoint_id': row['endpoint_id'], 'status_code': response['status_code']})
print(final)
~~~

### 왜 이 코드가 정답인지

이 답안은 문제에서 요구한 입력 구조를 먼저 읽고, 필요한 컬럼이나 selector만 선택한 뒤 결과를 명시적인 변수에 저장한다. `전체 허용 endpoint 실행` 단계에서 중요한 점은 출력 하나를 맞히는 것이 아니라 이후 문제에서 재사용할 수 있는 안정적인 중간 결과를 남기는 것이다. 빈칸에는 파일명, 컬럼명, 키 이름, 반복 제어 중 핵심 위치만 남겼기 때문에 학생이 데이터 구조를 이해해야 완성할 수 있다. 자주 보이는 오답은 컬럼명을 추측하거나, 중복과 실패 상태를 처리하지 않고 바로 저장하는 방식이다.

---

## 문제 13 정답 — 시도 로그 CSV 저장

~~~python
write_csv('lesson09_attempt_log.csv', all_logs, fieldnames=['endpoint_id', 'attempt_no', 'status_code', 'retry'])
print(Path('lesson09_attempt_log.csv').exists())
~~~

### 왜 이 코드가 정답인지

이 답안은 문제에서 요구한 입력 구조를 먼저 읽고, 필요한 컬럼이나 selector만 선택한 뒤 결과를 명시적인 변수에 저장한다. `시도 로그 CSV 저장` 단계에서 중요한 점은 출력 하나를 맞히는 것이 아니라 이후 문제에서 재사용할 수 있는 안정적인 중간 결과를 남기는 것이다. 빈칸에는 파일명, 컬럼명, 키 이름, 반복 제어 중 핵심 위치만 남겼기 때문에 학생이 데이터 구조를 이해해야 완성할 수 있다. 자주 보이는 오답은 컬럼명을 추측하거나, 중복과 실패 상태를 처리하지 않고 바로 저장하는 방식이다.

---

## 문제 14 정답 — 최종 요약 JSON 저장

~~~python
summary = {'target_count': len(allowed), 'attempt_count': len(all_logs), 'success_count': sum(1 for row in final if row['status_code'] == 200)}
Path('lesson09_retry_summary.json').write_text(json.dumps(summary, ensure_ascii=False, indent=2), encoding='utf-8')
print(summary)
~~~

### 왜 이 코드가 정답인지

이 답안은 문제에서 요구한 입력 구조를 먼저 읽고, 필요한 컬럼이나 selector만 선택한 뒤 결과를 명시적인 변수에 저장한다. `최종 요약 JSON 저장` 단계에서 중요한 점은 출력 하나를 맞히는 것이 아니라 이후 문제에서 재사용할 수 있는 안정적인 중간 결과를 남기는 것이다. 빈칸에는 파일명, 컬럼명, 키 이름, 반복 제어 중 핵심 위치만 남겼기 때문에 학생이 데이터 구조를 이해해야 완성할 수 있다. 자주 보이는 오답은 컬럼명을 추측하거나, 중복과 실패 상태를 처리하지 않고 바로 저장하는 방식이다.

---

## 문제 15 정답 — 운영 메모 문장 만들기

~~~python
templates = load_json('payload_templates.json')
message = templates['summary_template'].format(success=summary['success_count'], total=summary['target_count'], attempts=summary['attempt_count'])
print(message)
~~~

### 왜 이 코드가 정답인지

이 답안은 문제에서 요구한 입력 구조를 먼저 읽고, 필요한 컬럼이나 selector만 선택한 뒤 결과를 명시적인 변수에 저장한다. `운영 메모 문장 만들기` 단계에서 중요한 점은 출력 하나를 맞히는 것이 아니라 이후 문제에서 재사용할 수 있는 안정적인 중간 결과를 남기는 것이다. 빈칸에는 파일명, 컬럼명, 키 이름, 반복 제어 중 핵심 위치만 남겼기 때문에 학생이 데이터 구조를 이해해야 완성할 수 있다. 자주 보이는 오답은 컬럼명을 추측하거나, 중복과 실패 상태를 처리하지 않고 바로 저장하는 방식이다.

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
