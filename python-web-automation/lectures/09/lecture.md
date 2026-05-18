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

[![Open in Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/Kevin-innovation/jupyter-lecture/blob/main/python-web-automation/lectures/09/%5B%ED%95%99%EC%83%9D%EC%9A%A9%5D%20%EB%A0%88%EC%8A%A8%2009%20%E2%80%94%20%EC%86%8D%EB%8F%84%20%EC%A0%9C%ED%95%9C%2C%20%EC%9E%AC%EC%8B%9C%EB%8F%84%2C%20%EB%A1%9C%EA%B7%B8.ipynb)

> 코랩에서 실행하려면 우측 상단 또는 아래의 **Open in Colab** 버튼을 클릭한다. 이 노트북은 학생용 읽기와 따라하기 자료다.

이 노트북은 읽기와 따라하기용 강의 노트북이다. 자동화가 실패 응답을 만났을 때 무한 반복하지 않고, 요청 간격과 재시도 정책, 로그를 남기는 운영형 수집기를 안전한 합성 fixture로 연습한다.

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

이 셀은 수집 대상과 응답 계획을 분리해서 읽는다. 학생은 실제 서버 대신 합성 응답 계획으로 실패 상황을 안전하게 재현한다는 점을 이해해야 한다.

~~~python
targets = load_csv('targets.csv')
plan = load_csv('response_plan.csv')
print(len(targets), len(plan))
~~~
대상과 계획을 분리하면 같은 코드로 여러 실패 시나리오를 반복할 수 있다.

---

## 4. 자료 구조 확인

robots 샘플 파싱은 접근 가능한 경로를 먼저 확인하는 습관을 만들기 위한 단계다. 실제 표준 전체 구현보다 차단 경로를 발견하는 흐름에 집중한다.

~~~python
blocked = parse_disallow(load_text('robots_sample.txt'))
print(blocked)
~~~
차단 경로는 코드 실행 전에 제외해야 한다. 나중에 실패한 뒤 거르는 방식은 안전하지 않다.

---

## 5. 품질 기준 적용

허용 대상 필터링은 자동화의 안전 장치다. 차단 경로를 제외하고 난 뒤 어떤 endpoint만 남는지 확인하게 한다.

~~~python
allowed = [row for row in targets if not any(urlparse(row['url']).path.startswith(path) for path in blocked)]
print([row['endpoint_id'] for row in allowed])
~~~
필터 결과는 endpoint_id 목록으로 확인하면 빠르다. URL 전체보다 수업 중 비교하기 쉽다.

---

## 6. 저장과 보고

PlannedFetcher는 실제 요청을 보내지 않고 상태 코드 변화를 연습하게 한다. 같은 endpoint를 여러 번 호출했을 때 응답이 어떻게 달라지는지 관찰한다.

~~~python
fetcher = PlannedFetcher(plan)
print(fetcher.get('notice'))
print(fetcher.get('notice'))
~~~
합성 응답은 외부 서버 부하 없이 재시도 흐름을 보여준다.

---

## 7. 운영 관점 점검

상태 코드별 재시도 여부를 표처럼 출력하면 정책이 명확해진다. 학생은 404와 503을 같은 실패로 묶지 않아야 한다.

~~~python
for status in [200, 404, 429, 503]:
    print(status, should_retry(status))
~~~
재시도 정책은 함수로 분리해야 테스트하기 쉽다. 상태 코드가 추가되어도 한 곳만 바꾸면 된다.

---

## 8. 마무리 체크

backoff 지연 목록은 요청 간격이 점점 늘어나는 이유를 보여준다. 숫자보다 서버 부담을 줄이는 운영 목적을 먼저 설명한다.

~~~python
print([backoff_delay(i) for i in range(1, 6)])
~~~
backoff 값은 실제 sleep 없이도 정책을 검토할 수 있게 해준다.

---

## 9. 핵심 개념

재시도 함수는 마지막 응답과 시도 로그를 함께 반환한다. 성공 여부만 돌려주면 실패 원인을 잃기 때문에 로그 구조가 중요하다.

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
로그를 반환하면 성공과 실패 모두 같은 형식으로 저장할 수 있다.

---

## 10. 자료 구조 확인

시도 로그 저장은 운영 자동화의 증거를 남기는 과정이다. 파일을 만든 뒤에는 행 수와 컬럼을 확인하게 한다.

~~~python
write_csv('lesson09_attempt_log.csv', logs)
print(Path('lesson09_attempt_log.csv').exists())
~~~
CSV 로그는 수업 후에도 다시 열어볼 수 있는 실행 기록이다.

---

## 데이터 출처와 안전 규칙

targets.csv는 수집 대상의 우선순위와 최소 요청 간격을 담는다. response_plan.csv는 endpoint별 시도 순서와 상태 코드를 담은 합성 응답 계획이다. robots_sample.txt는 Disallow 규칙 해석 연습용이다. payload_templates.json은 로그 요약 문구와 상태 설명을 제공한다.

- 모든 파일은 수업용 합성 데이터다.
- 실제 사이트에 반복 요청하지 않는다.
- 저장 파일은 레슨 폴더 또는 코랩 현재 작업 폴더에만 만든다.
- 외부 사이트로 확장할 때는 약관, robots.txt, 요청 간격, 개인정보 포함 여부를 먼저 확인한다.

---

## 수업 운영 메모

이 절은 수업 중 교사가 질문으로 풀어낼 수 있는 운영형 설명이다. 학생이 셀을 실행한 뒤 결과만 맞히지 않고 자동화 절차를 말로 설명하도록 돕는다.

### 1. 실패를 정상 흐름에 포함

운영 자동화에서 실패는 예외적인 사건이 아니라 자주 발생하는 상태다. 429, 503, 404를 구분하고 각각 재시도, 대기, 중단 중 어떤 행동을 할지 정해야 한다.

### 2. 재시도 상한

재시도는 무한히 반복하면 안 된다. max_attempts를 두고 마지막 응답과 로그를 반환하게 해야 실패해도 프로그램이 멈추지 않고 보고서를 남긴다.

### 3. backoff 의미

지수 backoff는 서버가 회복할 시간을 주면서 클라이언트 요청도 줄이는 방법이다. 학생에게 숫자 공식보다 왜 첫 실패 이후 바로 반복 요청하면 안 되는지 먼저 설명한다.

### 4. robots 샘플 해석

이 수업의 robots_sample은 실제 표준 전체를 구현하려는 목적이 아니다. 자동화 전에 허용 경로와 차단 경로를 읽는 습관을 들이는 안전 장치다.

### 5. 우선순위와 간격

모든 endpoint를 같은 순서로 요청하지 않고 priority와 min_interval_seconds를 기준으로 계획을 만든다. 이 구조가 있으면 나중에 중요한 작업을 먼저 처리할 수 있다.

### 6. 로그 컬럼

로그에는 endpoint_id, attempt_no, status_code, retry 여부가 최소한 들어가야 한다. 누가 봐도 언제 어떤 상태에서 멈췄는지 알 수 있어야 한다.

### 7. 운영 메모

자동화 결과는 개발자만 보는 것이 아니다. 성공 건수, 전체 대상, 총 시도 횟수를 한 문장으로 요약하면 비개발 운영자도 상태를 파악할 수 있다.

---

## 실패를 정상 흐름으로 다루기

자동화 수업에서 가장 위험한 코드는 실패를 전혀 예상하지 않는 코드다. 한 번 요청해서 바로 성공하면 예제는 짧아 보이지만, 실제 운영에서는 속도 제한, 임시 장애, 차단 경로, 삭제된 URL이 계속 섞인다. 이번 레슨은 이런 상태를 예외 상황이 아니라 자동화 흐름의 일부로 다루는 연습이다.

학생에게는 상태 코드를 세 가지 행동으로 나누어 설명한다. 200은 성공이므로 결과를 저장한다. 429, 503처럼 잠시 후 회복될 수 있는 상태는 재시도한다. 403, 404처럼 권한이나 경로 문제가 명확한 상태는 즉시 중단한다. 이 기준이 없으면 코드가 무한 반복하거나, 반대로 잠깐 기다리면 될 요청을 너무 빨리 실패 처리하게 된다.

## 요청 간격과 backoff

요청 간격은 단순한 sleep이 아니라 상대 서버와 우리 서비스 모두를 보호하는 운영 규칙이다. targets.csv의 min_interval_seconds는 endpoint마다 최소한으로 기다려야 하는 시간을 표현한다. backoff_delay는 실패가 반복될수록 기다리는 시간을 늘려 서버 회복 시간을 확보한다. 수업에서는 실제 time.sleep을 길게 걸지 않고 숫자 목록으로 정책을 확인한다.

학생이 자주 하는 실수는 재시도를 빠르게 여러 번 반복하는 것이다. 이는 자동화가 실패했을 때 오히려 요청량을 늘리는 구조다. 로그에 attempt_no와 retry 값을 남기면 몇 번 시도했고 왜 멈췄는지 설명할 수 있다.

## robots 샘플을 먼저 읽는 이유

robots_sample.txt는 실제 robots.txt 표준 전체를 구현하려는 자료가 아니다. 자동화 전에 차단 경로를 읽고 제외하는 습관을 만들기 위한 안전 장치다. 이번 fixture에서는 /private과 /tmp 경로가 제외된다. 학생이 필터링을 제대로 했다면 admin과 media endpoint는 실행 대상에서 빠져야 한다.

실제 사이트로 확장할 때는 robots.txt만으로 충분하지 않다. 서비스 약관, 로그인 여부, 개인정보 포함 여부, 요청량 제한을 함께 확인해야 한다. 이번 레슨에서는 그중 “시작 전에 허용 범위를 확인한다”는 첫 습관을 코드로 만든다.

## 로그 설계 기준

로그는 실패를 숨기지 않기 위해 남긴다. 최소 컬럼은 endpoint_id, attempt_no, status_code, retry이다. endpoint_id는 어떤 대상인지, attempt_no는 몇 번째 시도인지, status_code는 어떤 결과인지, retry는 다시 시도할 판단인지 보여준다. 이 네 가지가 있으면 수업 후에도 자동화가 어디서 멈췄는지 추적할 수 있다.

최종 요약에는 target_count, attempt_count, success_count가 들어간다. 성공 수만 보면 전체 규모를 알 수 없고, 전체 대상 수만 보면 재시도가 얼마나 있었는지 알 수 없다. 세 값을 함께 봐야 운영자가 “정상 완료”, “일부 실패”, “요청량 과다”를 구분할 수 있다.

## 교사가 확인할 질문

- 404와 503을 왜 다르게 처리해야 하는가?
- retry=True가 붙은 로그는 어떤 상태를 의미하는가?
- max_attempts가 없으면 어떤 문제가 생기는가?
- /private, /tmp 경로가 실행 대상에서 빠졌는가?
- attempt_count가 target_count보다 커질 수 있는 이유는 무엇인가?

이 질문에 답할 수 있으면 학생은 단순 요청 코드가 아니라 운영 가능한 재시도 정책을 이해한 것이다.


## 상태 코드별 운영 판단

상태 코드는 숫자 하나처럼 보이지만 자동화의 다음 행동을 결정하는 신호다. 200은 정상 처리이므로 결과를 저장하고 다음 endpoint로 넘어간다. 429는 요청이 너무 빠르다는 의미로 해석할 수 있으므로 바로 반복하지 않고 backoff를 적용한다. 503은 서버가 일시적으로 처리하지 못하는 상태일 수 있으므로 일정 횟수까지 재시도한다. 404는 경로가 없다는 뜻이므로 같은 주소를 즉시 다시 요청해도 해결될 가능성이 낮다.

이 구분이 있어야 자동화가 예의 있게 동작한다. 모든 실패를 재시도하면 서버에 부담을 주고, 모든 실패를 중단하면 일시 장애를 회복할 기회를 잃는다. 학생에게는 “상태 코드가 같지 않으면 다음 행동도 같지 않다”는 원칙을 반복해서 확인시킨다.

## PlannedFetcher를 쓰는 이유

실제 사이트를 대상으로 재시도 수업을 하면 의도하지 않게 많은 요청을 보낼 수 있다. PlannedFetcher는 response_plan.csv에 적힌 순서대로 상태 코드를 돌려주기 때문에 외부 서버 없이도 실패, 재시도, 성공 흐름을 재현한다. 이 구조 덕분에 notice는 429 뒤 200, status는 503 뒤 503 뒤 200처럼 수업자가 설계한 흐름을 안정적으로 보여줄 수 있다.

학생이 PlannedFetcher를 이해하면 테스트 가능한 자동화의 개념도 자연스럽게 배운다. 실제 requests.get을 직접 호출하는 코드보다, 응답을 주입할 수 있는 구조가 안전하고 검증하기 쉽다. 이후 실전 코드로 확장할 때도 요청 함수와 처리 함수를 분리하면 테스트가 쉬워진다.

## 로그를 먼저 설계하는 습관

재시도 로직을 작성할 때는 성공 결과보다 로그 구조를 먼저 설계하는 편이 좋다. 어떤 endpoint를 몇 번째 시도했고, 어떤 status_code를 받았고, 그 상태를 retry로 판단했는지 남기면 나중에 원인을 추적할 수 있다. 로그가 없는 자동화는 실패했을 때 “안 됐다” 외에는 설명할 수 없다.

수업에서는 로그 행 하나를 다음처럼 읽게 한다.

- endpoint_id: 어떤 대상을 요청했는가?
- attempt_no: 몇 번째 시도였는가?
- status_code: 서버 또는 fixture가 어떤 응답을 냈는가?
- retry: 이 응답을 다시 시도할 대상으로 판단했는가?

이 네 값이 자연스럽게 읽히면 학생은 재시도 코드의 목적을 이해한 것이다.


## 수업 마무리 점검

마지막에는 학생에게 로그 한 줄을 직접 읽게 한다. 예를 들어 status endpoint의 두 번째 시도에서 503이 나왔다면 “status를 두 번째로 요청했고, 아직 재시도 가능한 상태라 다음 시도로 넘어간다”라고 말할 수 있어야 한다. 이 설명이 가능하면 코드가 단순 출력이 아니라 운영 기록이라는 점을 이해한 것이다.

또한 summary 문장을 만들 때 성공 수만 강조하지 않게 한다. “성공 3건”보다 “허용 대상 4건 중 3건 성공, 총 7회 시도”처럼 전체 규모와 요청량을 함께 말해야 한다. 자동화의 품질은 성공 여부뿐 아니라 얼마나 무리 없이 실행되었는지까지 포함한다.
