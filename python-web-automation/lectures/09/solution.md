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

[![Open in Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/Kevin-innovation/jupyter-lecture/blob/main/python-web-automation/lectures/09/%EB%A0%88%EC%8A%A8%2009%20%E2%80%94%20%EC%86%8D%EB%8F%84%20%EC%A0%9C%ED%95%9C%2C%20%EC%9E%AC%EC%8B%9C%EB%8F%84%2C%20%EB%A1%9C%EA%B7%B8.ipynb)

> 선생님용 강의 노트북이다. 코랩에서 확인하려면 우측 상단 또는 아래의 **Open in Colab** 버튼을 클릭한다.

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

targets.csv는 자동화가 접근할 후보 목록이다. 먼저 전체 대상을 변수 targets에 남겨야 robots 필터, priority 정렬, interval 변환을 이어서 처리할 수 있다. 행 수 출력은 입력이 정상적으로 읽혔는지 확인하는 가장 빠른 점검이다.

---

## 문제 2 정답 — 응답 계획 읽기

~~~python
plan = load_csv('response_plan.csv')
print(plan[0]['endpoint_id'])
~~~

### 왜 이 코드가 정답인지

response_plan.csv는 실제 서버 요청 대신 사용할 합성 응답 순서다. endpoint_id를 확인하면 어떤 대상의 응답 계획인지 알 수 있고, PlannedFetcher가 이 키를 기준으로 상태 코드를 순서대로 반환한다.

---

## 문제 3 정답 — robots 차단 경로 파싱

~~~python
blocked = parse_disallow(load_text('robots_sample.txt'))
print(blocked)
~~~

### 왜 이 코드가 정답인지

robots_sample.txt를 먼저 읽어 차단 경로를 추출해야 요청 전에 제외 대상을 알 수 있다. parse_disallow는 Disallow 줄만 모아 /private, /tmp 같은 경로 목록을 만들며, 이 목록은 다음 필터링의 기준이 된다.

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

허용 대상 필터링은 url path가 차단 경로로 시작하는지 확인한다. admin과 media처럼 차단 경로에 있는 endpoint를 실행하지 않는 것이 핵심이며, 실패한 뒤 제외하는 방식은 안전하지 않다.

---

## 문제 5 정답 — PlannedFetcher 첫 응답 확인

~~~python
fetcher = PlannedFetcher(plan)
response = fetcher.get('notice')
print(response['status_code'])
~~~

### 왜 이 코드가 정답인지

PlannedFetcher는 외부 요청 없이 상태 코드 흐름을 재현한다. plan을 넘겨 fetcher를 만들고 notice의 첫 응답 status_code를 확인하면 429 같은 실패 상태를 안전하게 실습할 수 있다.

---

## 문제 6 정답 — 재시도 상태 코드 판정

~~~python
for status in [200, 404, 429, 503]:
    print(status, should_retry(status))
~~~

### 왜 이 코드가 정답인지

should_retry는 상태 코드 정책을 한 곳에 모은 함수다. 429와 503은 재시도 대상이지만 404는 즉시 중단해야 하므로, 상태별 행동을 함수로 분리해야 이후 정책 변경이 쉽다.

---

## 문제 7 정답 — backoff 지연 목록 만들기

~~~python
delays = [backoff_delay(i) for i in range(1, 6)]
print(delays)
~~~

### 왜 이 코드가 정답인지

backoff_delay는 시도 횟수가 늘수록 기다리는 시간이 커지는 정책을 보여준다. range(1, 6)은 1회부터 5회까지의 지연을 만들며, 실제 sleep 없이도 요청량 제어 전략을 확인할 수 있다.

---

## 문제 8 정답 — 우선순위 순서 정렬

~~~python
ordered = sorted(allowed, key=lambda row: int(row['priority']))
print([row['endpoint_id'] for row in ordered])
~~~

### 왜 이 코드가 정답인지

priority는 어떤 endpoint를 먼저 처리할지 결정한다. 문자열로 읽힌 값을 int로 바꾸어 정렬해야 10 같은 값이 2보다 앞서는 문자열 정렬 오류를 피할 수 있다.

---

## 문제 9 정답 — 요청 간격 숫자 변환

~~~python
intervals = {row['endpoint_id']: float(row['min_interval_seconds']) for row in allowed}
print(intervals)
~~~

### 왜 이 코드가 정답인지

min_interval_seconds는 CSV에서 문자열로 읽히므로 float로 변환해야 계산에 사용할 수 있다. endpoint_id를 key로 만든 dict는 나중에 endpoint별 대기 정책을 조회하기 쉽다.

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

단일 endpoint 루프는 성공하거나 재시도 불가 상태가 나오면 break로 멈춘다. 이 조건이 없으면 이미 성공했거나 404처럼 멈춰야 하는 상태에서도 불필요하게 계속 시도할 수 있다.

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

fetch_with_retry는 마지막 response와 시도 logs를 함께 반환한다. 성공 여부만 반환하면 실패 원인을 잃으므로, 운영 자동화에서는 모든 시도를 기록한 로그가 필수다.

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

전체 허용 endpoint 실행은 ordered 목록만 대상으로 한다. robots에서 제외한 endpoint를 실행하지 않고, 각 endpoint의 최종 status_code와 전체 시도 로그를 분리해 저장 준비를 한다.

---

## 문제 13 정답 — 시도 로그 CSV 저장

~~~python
write_csv('lesson09_attempt_log.csv', all_logs, fieldnames=['endpoint_id', 'attempt_no', 'status_code', 'retry'])
print(Path('lesson09_attempt_log.csv').exists())
~~~

### 왜 이 코드가 정답인지

시도 로그 CSV는 운영자가 실패 과정을 다시 볼 수 있는 증거다. endpoint_id, attempt_no, status_code, retry 컬럼을 명시하면 로그 행이 비어 있거나 컬럼 순서가 흔들리는 문제를 줄일 수 있다.

---

## 문제 14 정답 — 최종 요약 JSON 저장

~~~python
summary = {'target_count': len(allowed), 'attempt_count': len(all_logs), 'success_count': sum(1 for row in final if row['status_code'] == 200)}
Path('lesson09_retry_summary.json').write_text(json.dumps(summary, ensure_ascii=False, indent=2), encoding='utf-8')
print(summary)
~~~

### 왜 이 코드가 정답인지

최종 요약 JSON은 전체 대상 수, 총 시도 횟수, 성공 수를 함께 남긴다. 200 상태만 성공으로 세어야 하고, attempt_count는 재시도 때문에 target_count보다 커질 수 있다.

---

## 문제 15 정답 — 운영 메모 문장 만들기

~~~python
templates = load_json('payload_templates.json')
message = templates['summary_template'].format(success=summary['success_count'], total=summary['target_count'], attempts=summary['attempt_count'])
print(message)
~~~

### 왜 이 코드가 정답인지

payload_templates.json에서 문장 템플릿을 읽으면 운영 메모 문구를 코드 밖에서 관리할 수 있다. success, total, attempts 값을 채워 넣으면 비개발자도 읽을 수 있는 요약 문장이 된다.

## 교사용 검토 메모

이번 정답지는 재시도 정책과 로그 구조를 확인하기 위한 기준이다. 학생 답안이 모범 코드와 조금 달라도 다음 조건을 만족하면 인정할 수 있다.

- robots 차단 경로를 실행 대상에서 제외했다.
- 재시도 대상 상태 코드와 즉시 중단 상태 코드를 구분했다.
- max_attempts로 무한 반복을 막았다.
- 각 시도마다 endpoint_id, attempt_no, status_code를 로그로 남겼다.
- 최종 응답과 시도 로그를 함께 반환했다.
- CSV 로그와 JSON 요약을 만들고 존재 여부를 확인했다.

반대로 출력값이 우연히 맞아도 retry 정책이 코드에 드러나지 않거나, 실패를 로그로 남기지 않았다면 보완하게 한다. 운영 자동화에서는 성공한 데이터보다 실패했을 때 남는 기록이 더 중요하다.

## 문제별 채점 포인트

### 문제 1 채점 포인트

targets.csv를 정확히 읽고 targets 변수에 저장했는지 확인한다. targets는 이후 robots 필터와 정렬의 입력이므로 행 수 출력만 맞히고 변수를 남기지 않으면 뒤 문제가 불안정해진다.

### 문제 2 채점 포인트

response_plan.csv는 외부 서버 대신 실패 상황을 재현하는 핵심 fixture다. endpoint_id 컬럼을 확인하지 못하면 PlannedFetcher의 기준 키를 이해하지 못한 것이다.

### 문제 3 채점 포인트

robots 샘플에서 Disallow 경로만 추출하는지 확인한다. Allow와 Crawl-delay까지 모두 섞어 blocked에 넣으면 허용 대상 필터가 틀어진다.

### 문제 4 채점 포인트

urlparse(row['url']).path를 사용해 경로만 비교하는지 본다. 전체 URL 문자열에 단순 포함 검사를 하면 도메인이나 쿼리 문자열 때문에 의도하지 않은 결과가 나올 수 있다.

### 문제 5 채점 포인트

PlannedFetcher를 plan으로 초기화해야 한다. 첫 응답을 확인하는 문제이므로 실제 requests.get을 호출하면 안 된다. 이 코스는 합성 응답으로 안전하게 흐름을 검증한다.

### 문제 6 채점 포인트

상태 코드별 정책이 함수로 분리되어야 한다. 200, 404, 429, 503의 결과를 비교하면 학생이 재시도와 중단을 구분했는지 빠르게 볼 수 있다.

### 문제 7 채점 포인트

range 끝값을 이해했는지 확인한다. 1~5회 시도를 만들려면 range(1, 6)이 필요하다. backoff 결과가 cap을 넘지 않는지도 함께 볼 수 있다.

### 문제 8 채점 포인트

priority는 문자열이므로 int 변환이 필요하다. 작은 fixture에서는 문제가 드러나지 않아도 운영 데이터에서 priority 10이 생기면 문자열 정렬 오류가 바로 나타난다.

### 문제 9 채점 포인트

요청 간격은 숫자로 변환해야 이후 sleep이나 스케줄 계산에 사용할 수 있다. endpoint_id를 key로 만든 dict 구조가 가장 단순하고 재사용하기 좋다.

### 문제 10 채점 포인트

break 위치가 중요하다. 로그를 남긴 뒤 성공 또는 재시도 불가 상태를 만나면 멈춰야 한다. 로그를 남기기 전에 멈추면 마지막 상태가 기록되지 않는다.

### 문제 11 채점 포인트

함수는 response와 logs를 함께 반환해야 한다. 마지막 response만 있으면 시도 과정을 잃고, logs만 있으면 최종 body나 status를 다루기 어렵다.

### 문제 12 채점 포인트

ordered는 robots 필터 후 priority 정렬된 목록이어야 한다. 차단 endpoint가 final에 포함되면 안전 필터가 실패한 것이다.

### 문제 13 채점 포인트

로그 CSV는 fieldnames를 명시하는 편이 안전하다. retry 컬럼이 누락되면 운영자가 다시 시도해야 했던 요청을 구분할 수 없다.

### 문제 14 채점 포인트

success_count는 200만 세어야 한다. 404나 403처럼 정상적으로 멈춘 상태도 성공으로 세면 운영 보고가 왜곡된다.

### 문제 15 채점 포인트

템플릿을 JSON에서 읽고 format으로 값을 채우는지 확인한다. 문구를 코드에 직접 적어도 결과는 비슷하지만, 운영 문구를 데이터 파일로 분리하는 연습이 줄어든다.

## 예상 출력 흐름

허용 대상은 robots 차단 경로를 제외한 public endpoint만 남는다. notice는 첫 시도 429 후 두 번째 시도에서 성공하고, status는 503을 두 번 거친 뒤 성공한다. report와 archive도 각각 계획된 응답을 반환한다. 이 흐름 때문에 attempt_count는 target_count보다 커진다.

archive는 404를 반환하므로 재시도하지 않고 멈춰야 한다. 이 상태는 “프로그램 오류”가 아니라 “재시도해도 해결되지 않을 가능성이 큰 상태”로 설명한다. 학생이 404를 재시도 대상으로 넣으면 불필요한 요청이 늘어난다는 점을 짚어준다.

## 수업 마무리 피드백 예시

좋은 답안은 실패를 숨기지 않는다. “허용 대상만 실행했고, 429와 503은 재시도했으며, 404는 중단하고 로그에 남겼다”라고 설명할 수 있으면 핵심을 이해한 것이다. 보완이 필요한 답안은 대체로 성공 결과만 출력하고 시도 로그를 잃어버린다. 이때는 최종 응답보다 all_logs를 먼저 출력하게 하는 편이 효과적이다.


## 단계별 해설 보충

### 입력 파일 네 가지의 역할

targets.csv는 실행 후보 목록이고, response_plan.csv는 합성 응답 순서다. robots_sample.txt는 실행 전에 제외해야 할 경로를 알려주고, payload_templates.json은 운영자가 읽을 문장을 만든다. 네 파일의 역할을 구분해야 학생이 특정 CSV 하나만 읽고 끝내지 않는다. 이번 레슨의 자동화는 요청 후보, 안전 기준, 응답 계획, 보고 문구가 분리되어 있어야 재실행과 수정이 쉽다.

### robots 필터가 먼저 나오는 이유

robots 필터는 요청 후 처리하는 단계가 아니라 요청 전에 실행 대상을 줄이는 단계다. /private/admin과 /tmp/media는 response_plan에 있더라도 실행하지 않아야 한다. 학생이 필터를 뒤로 미루면 차단 경로를 이미 요청한 뒤에야 제외하는 구조가 되므로 안전하지 않다. 이 기준은 실제 사이트에서도 매우 중요하다.

### 재시도 가능 상태와 중단 상태

429와 503은 일시적인 상황일 수 있으므로 제한된 횟수 안에서 재시도한다. 404는 경로가 없다는 신호이므로 같은 주소를 반복해도 해결되지 않는다. 403은 권한 문제이므로 자동화가 임의로 우회하려고 하면 안 된다. 상태 코드 정책은 조직마다 다를 수 있지만, 정책이 함수로 분리되어 있어야 수업과 운영에서 설명이 가능하다.

### max_attempts의 의미

max_attempts는 자동화의 브레이크다. 실패가 반복될 때 멈추는 기준이 없으면 무한 루프나 과도한 요청이 된다. 이 값은 정답을 맞히기 위한 숫자가 아니라 운영 리스크를 제한하는 장치다. 학생이 이 의미를 이해하면 while True로 요청을 반복하는 코드를 피하게 된다.

### backoff와 interval의 차이

min_interval_seconds는 정상 요청 사이의 최소 간격이고, backoff는 실패가 발생했을 때 늘어나는 대기 시간이다. 둘 다 기다림을 표현하지만 목적이 다르다. interval은 평상시 속도 제한, backoff는 장애나 제한에 대한 회복 전략이다. 학생이 두 개념을 구분하면 실제 자동화에서 요청량을 더 안정적으로 조절할 수 있다.

### all_logs와 final을 나누는 이유

all_logs는 모든 시도 기록이다. final은 endpoint별 마지막 상태만 모은 요약이다. 둘을 하나로 합치면 상세 추적과 최종 보고가 섞인다. 운영자는 보통 final을 먼저 보고, 문제가 있는 endpoint만 all_logs를 확인한다. 이 구분은 8강에서 checked와 clean_rows를 나눈 흐름과도 연결된다.

### 저장 후 확인

lesson09_attempt_log.csv와 lesson09_retry_summary.json은 단순 출력물이 아니라 검수 대상이다. 파일이 존재하는지 확인하고, 필요하면 다시 읽어 행 수와 key를 확인한다. 자동화는 실행 중 화면에 출력된 값보다 저장된 결과가 더 오래 남는다. 그래서 저장 후 확인은 마지막 셀의 필수 습관으로 다룬다.

## 오답 유형과 피드백 문장

- 차단 경로까지 실행한 경우: “요청 전에 허용 범위를 먼저 줄여야 한다. robots 필터가 실행 목록보다 앞에 있어야 한다.”
- 모든 실패를 재시도한 경우: “404와 403은 같은 주소를 반복해도 해결되지 않는다. 재시도 가능 상태와 중단 상태를 분리해야 한다.”
- 로그 없이 최종 결과만 남긴 경우: “운영자는 왜 실패했는지 알아야 하므로 attempt_no와 status_code가 있는 로그가 필요하다.”
- max_attempts가 없는 경우: “실패가 반복될 때 멈출 기준이 없으면 자동화가 서버에 부담을 줄 수 있다.”
- retry 컬럼이 없는 경우: “상태 코드를 어떤 정책으로 판단했는지 로그에서 바로 볼 수 있어야 한다.”

## 재실행 검증 기준

선생님용 노트북은 커널을 재시작한 뒤 위에서 아래로 실행해도 같은 결과가 나와야 한다. PlannedFetcher는 내부 cursor를 사용하므로 전체 실행마다 새 인스턴스를 만들어야 한다. 이미 사용한 fetcher를 재사용하면 같은 endpoint의 첫 응답이 달라질 수 있다. 학생 답안에서 결과가 이상하면 fetcher를 언제 새로 만들었는지 먼저 확인한다.

CSV와 JSON 파일은 같은 이름으로 덮어쓴다. 수업 중 여러 번 실행해도 파일 이름이 늘어나지 않아야 한다. 만약 실행할 때마다 다른 파일명이 생기면 학생이 결과 비교를 하기 어려워진다. 운영 자동화에서도 산출물 이름 규칙은 일정해야 한다.

## 예상 출력 확인

robots 필터 후 허용 대상은 public 경로만 남아야 한다. notice는 429 후 200으로 끝나고, status는 503 두 번 뒤 200으로 끝난다. report는 한 번에 200이고, archive는 404로 중단된다. 따라서 성공 수는 전체 허용 대상 수보다 작을 수 있고, 총 시도 횟수는 허용 대상 수보다 클 수 있다.

이 숫자 관계를 학생이 설명할 수 있으면 재시도 정책을 이해한 것이다. 성공 수만 맞히고 attempt_count가 왜 커졌는지 설명하지 못하면 로그를 읽지 않은 상태다.

## 교사용 빠른 판정표

| 항목 | 통과 기준 | 다시 볼 신호 |
| --- | --- | --- |
| 대상 읽기 | targets와 plan을 별도로 읽는다 | 한 파일만 읽고 끝낸다 |
| 안전 필터 | /private, /tmp 경로를 제외한다 | 차단 endpoint가 final에 들어간다 |
| 상태 정책 | 429/503 재시도, 404 중단 | 모든 실패를 같은 방식으로 처리한다 |
| backoff | 시도 횟수별 지연을 계산한다 | 재시도 사이의 정책이 없다 |
| 로그 | endpoint_id, attempt_no, status_code, retry 기록 | 최종 결과만 출력한다 |
| 요약 | target_count, attempt_count, success_count 저장 | 성공 수만 남긴다 |
| 재실행 | 새 fetcher로 같은 결과를 만든다 | 이전 cursor 상태에 의존한다 |


## 수업 중 디버깅 순서

학생 코드가 예상과 다르면 먼저 allowed 목록을 확인한다. 차단 endpoint가 남아 있으면 robots 필터가 잘못된 것이다. 그 다음 ordered 목록을 확인해 priority 정렬이 의도대로 되었는지 본다. 이후 fetch_with_retry를 단일 endpoint에 적용해 response와 logs가 함께 나오는지 확인한다. 마지막으로 전체 실행에서 all_logs 길이와 final 길이를 비교한다.

이 순서를 지키면 대부분의 오류를 빠르게 찾을 수 있다. 처음부터 최종 JSON만 보면 어떤 단계에서 문제가 생겼는지 알기 어렵다. 자동화 수업의 피드백은 결과 파일보다 중간 리스트를 따라가는 방식이 더 정확하다.
