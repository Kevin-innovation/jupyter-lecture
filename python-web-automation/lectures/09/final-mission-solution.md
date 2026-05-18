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

# 레슨 09 — 최종 미션 모범 답안

[![Open in Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/Kevin-innovation/jupyter-lecture/blob/main/python-web-automation/lectures/09/%EB%A0%88%EC%8A%A8%2009%20%E2%80%94%20%EC%86%8D%EB%8F%84%20%EC%A0%9C%ED%95%9C%2C%20%EC%9E%AC%EC%8B%9C%EB%8F%84%2C%20%EB%A1%9C%EA%B7%B8.ipynb)

> 선생님용 최종 미션 모범 답안이다. 코랩에서 확인하려면 우측 상단 또는 아래의 **Open in Colab** 버튼을 클릭한다.

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

targets = load_csv('targets.csv')
print(len(targets))

plan = load_csv('response_plan.csv')
print(plan[0]['endpoint_id'])

blocked = parse_disallow(load_text('robots_sample.txt'))
print(blocked)

allowed = []
for row in targets:
    path = urlparse(row['url']).path
    if not any(path.startswith(rule) for rule in blocked):
        allowed.append(row)
print([row['endpoint_id'] for row in allowed])

fetcher = PlannedFetcher(plan)
response = fetcher.get('notice')
print(response['status_code'])

for status in [200, 404, 429, 503]:
    print(status, should_retry(status))

delays = [backoff_delay(i) for i in range(1, 6)]
print(delays)

ordered = sorted(allowed, key=lambda row: int(row['priority']))
print([row['endpoint_id'] for row in ordered])

intervals = {row['endpoint_id']: float(row['min_interval_seconds']) for row in allowed}
print(intervals)

fetcher = PlannedFetcher(plan)
logs = []
for attempt_no in range(1, 4):
    response = fetcher.get('status')
    logs.append({'endpoint_id': 'status', 'attempt_no': attempt_no, 'status_code': response['status_code']})
    if response['status_code'] == 200 or not should_retry(response['status_code']):
        break
print(logs)

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

fetcher = PlannedFetcher(plan)
all_logs = []
final = []
for row in ordered:
    response, logs = fetch_with_retry(fetcher, row['endpoint_id'])
    all_logs.extend(logs)
    final.append({'endpoint_id': row['endpoint_id'], 'status_code': response['status_code']})
print(final)

write_csv('lesson09_attempt_log.csv', all_logs, fieldnames=['endpoint_id', 'attempt_no', 'status_code', 'retry'])
print(Path('lesson09_attempt_log.csv').exists())

summary = {'target_count': len(allowed), 'attempt_count': len(all_logs), 'success_count': sum(1 for row in final if row['status_code'] == 200)}
Path('lesson09_retry_summary.json').write_text(json.dumps(summary, ensure_ascii=False, indent=2), encoding='utf-8')
print(summary)

templates = load_json('payload_templates.json')
message = templates['summary_template'].format(success=summary['success_count'], total=summary['target_count'], attempts=summary['attempt_count'])
print(message)
~~~

## 채점 메모

- 입력 파일을 모두 읽었는지 확인한다.
- 검증 기준이 코드에 명시되어 있는지 확인한다.
- 저장 파일과 운영 요약이 함께 있는지 확인한다.
