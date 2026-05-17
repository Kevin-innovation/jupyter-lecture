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

# 레슨 09 — 속도 제한, 재시도, 로그

[![Open in Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/Kevin-innovation/jupyter-lecture/blob/main/python-web-automation/lectures/09/[학생용] 레슨 09 — 속도 제한, 재시도, 로그.ipynb)

이 노트북은 읽기와 따라하기용 강의 노트북이다. 자동화가 실패 응답을 만났을 때 무한 반복하지 않고, 요청 간격과 재시도 정책, 로그를 남기는 운영형 수집기을 안전한 합성 fixture로 연습한다.

## 학습 목표

1. 재시도해야 하는 상태 코드와 즉시 중단할 상태 코드를 구분한다.
2. 지수 backoff와 최대 재시도 횟수를 코드로 표현한다.
3. robots 샘플 규칙과 요청 간격을 자동화 흐름에 반영한다.
4. 시도별 로그와 최종 요약을 CSV/JSON으로 저장한다.
5. 운영 중 실패를 숨기지 않고 보고서로 남긴다.

---

## 1. 수업 맥락과 안전 기준

실제 운영 자동화는 한 번에 성공하지 않는다. 네트워크 지연, 임시 장애, 속도 제한이 섞인다. 이 레슨은 외부 사이트에 반복 요청을 보내지 않고 계획된 응답 fixture로 실패와 회복을 안전하게 연습한다.

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

## 3. 핵심 개념

아래 셀은 강사가 먼저 실행 흐름을 보여주고, 학생이 같은 구조를 자기 말로 설명하도록 만든 예제다. 코드는 짧지만 입력, 처리, 출력이 분리되어 있어 나중에 문제 풀이로 확장하기 쉽다.

~~~python
targets = load_csv('targets.csv')
plan = load_csv('response_plan.csv')
print(len(targets), len(plan))
~~~
이 셀에서 확인해야 할 것은 출력값 하나가 아니라 데이터가 어떤 단계에서 바뀌었는지다. 자동화 수업에서는 중간 변수명을 명확히 두면 학생이 오류 위치를 쉽게 찾을 수 있다.

---

## 4. 자료 구조 확인

아래 셀은 강사가 먼저 실행 흐름을 보여주고, 학생이 같은 구조를 자기 말로 설명하도록 만든 예제다. 코드는 짧지만 입력, 처리, 출력이 분리되어 있어 나중에 문제 풀이로 확장하기 쉽다.

~~~python
blocked = parse_disallow(load_text('robots_sample.txt'))
print(blocked)
~~~
이 셀에서 확인해야 할 것은 출력값 하나가 아니라 데이터가 어떤 단계에서 바뀌었는지다. 자동화 수업에서는 중간 변수명을 명확히 두면 학생이 오류 위치를 쉽게 찾을 수 있다.

---

## 5. 품질 기준 적용

아래 셀은 강사가 먼저 실행 흐름을 보여주고, 학생이 같은 구조를 자기 말로 설명하도록 만든 예제다. 코드는 짧지만 입력, 처리, 출력이 분리되어 있어 나중에 문제 풀이로 확장하기 쉽다.

~~~python
allowed = [row for row in targets if not any(urlparse(row['url']).path.startswith(path) for path in blocked)]
print([row['endpoint_id'] for row in allowed])
~~~
이 셀에서 확인해야 할 것은 출력값 하나가 아니라 데이터가 어떤 단계에서 바뀌었는지다. 자동화 수업에서는 중간 변수명을 명확히 두면 학생이 오류 위치를 쉽게 찾을 수 있다.

---

## 6. 저장과 보고

아래 셀은 강사가 먼저 실행 흐름을 보여주고, 학생이 같은 구조를 자기 말로 설명하도록 만든 예제다. 코드는 짧지만 입력, 처리, 출력이 분리되어 있어 나중에 문제 풀이로 확장하기 쉽다.

~~~python
fetcher = PlannedFetcher(plan)
print(fetcher.get('notice'))
print(fetcher.get('notice'))
~~~
이 셀에서 확인해야 할 것은 출력값 하나가 아니라 데이터가 어떤 단계에서 바뀌었는지다. 자동화 수업에서는 중간 변수명을 명확히 두면 학생이 오류 위치를 쉽게 찾을 수 있다.

---

## 7. 운영 관점 점검

아래 셀은 강사가 먼저 실행 흐름을 보여주고, 학생이 같은 구조를 자기 말로 설명하도록 만든 예제다. 코드는 짧지만 입력, 처리, 출력이 분리되어 있어 나중에 문제 풀이로 확장하기 쉽다.

~~~python
for status in [200, 404, 429, 503]:
    print(status, should_retry(status))
~~~
이 셀에서 확인해야 할 것은 출력값 하나가 아니라 데이터가 어떤 단계에서 바뀌었는지다. 자동화 수업에서는 중간 변수명을 명확히 두면 학생이 오류 위치를 쉽게 찾을 수 있다.

---

## 8. 마무리 체크

아래 셀은 강사가 먼저 실행 흐름을 보여주고, 학생이 같은 구조를 자기 말로 설명하도록 만든 예제다. 코드는 짧지만 입력, 처리, 출력이 분리되어 있어 나중에 문제 풀이로 확장하기 쉽다.

~~~python
print([backoff_delay(i) for i in range(1, 6)])
~~~
이 셀에서 확인해야 할 것은 출력값 하나가 아니라 데이터가 어떤 단계에서 바뀌었는지다. 자동화 수업에서는 중간 변수명을 명확히 두면 학생이 오류 위치를 쉽게 찾을 수 있다.

---

## 9. 핵심 개념

아래 셀은 강사가 먼저 실행 흐름을 보여주고, 학생이 같은 구조를 자기 말로 설명하도록 만든 예제다. 코드는 짧지만 입력, 처리, 출력이 분리되어 있어 나중에 문제 풀이로 확장하기 쉽다.

~~~python
def fetch_with_retry(fetcher, endpoint_id, max_attempts=3):
    logs = []
    for attempt in range(1, max_attempts + 1):
        response = fetcher.get(endpoint_id)
        logs.append({'endpoint_id': endpoint_id, 'attempt_no': attempt, 'status_code': response['status_code'], 'retry': should_retry(response['status_code'])})
        if response['status_code'] == 200 or not should_retry(response['status_code']):
            return response, logs
    return response, logs
response, logs = fetch_with_retry(PlannedFetcher(plan), 'status')
print(response)
print(logs)
~~~
이 셀에서 확인해야 할 것은 출력값 하나가 아니라 데이터가 어떤 단계에서 바뀌었는지다. 자동화 수업에서는 중간 변수명을 명확히 두면 학생이 오류 위치를 쉽게 찾을 수 있다.

---

## 10. 자료 구조 확인

아래 셀은 강사가 먼저 실행 흐름을 보여주고, 학생이 같은 구조를 자기 말로 설명하도록 만든 예제다. 코드는 짧지만 입력, 처리, 출력이 분리되어 있어 나중에 문제 풀이로 확장하기 쉽다.

~~~python
write_csv('lesson09_attempt_log.csv', logs)
print(Path('lesson09_attempt_log.csv').exists())
~~~
이 셀에서 확인해야 할 것은 출력값 하나가 아니라 데이터가 어떤 단계에서 바뀌었는지다. 자동화 수업에서는 중간 변수명을 명확히 두면 학생이 오류 위치를 쉽게 찾을 수 있다.

---

## 데이터 출처와 안전 규칙

targets.csv는 수집 대상의 우선순위와 최소 요청 간격을 담는다. response_plan.csv는 endpoint별 시도 순서와 상태 코드를 담은 합성 응답 계획이다. robots_sample.txt는 Disallow 규칙 해석 연습용이다. payload_templates.json은 로그 요약 문구와 상태 설명을 제공한다.

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
