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

## 강의 보강 노트

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

### 체크포인트 1: 출력 형태 표준화한다

실제 사이트 확장 전에 위험을 낮추기 위해, 레슨 09 속도 제한, 재시도, 로그에서는 출력 형태을/를 표준화한다는 과정을 별도의 단계로 둔다. 이때 학생은 화면에 보이는 값 하나보다 어떤 입력에서 어떤 변환을 거쳐 결과가 나왔는지 설명해야 한다. 강사는 변수명, 출력 형태, 저장 여부를 함께 확인하며 다음 실행에서도 같은 결과가 나오는지 묻는다.

### 체크포인트 2: 반복 단위 검증한다

저장 파일의 신뢰도를 높이기 위해, 레슨 09 속도 제한, 재시도, 로그에서는 반복 단위을/를 검증한다는 과정을 별도의 단계로 둔다. 이때 학생은 화면에 보이는 값 하나보다 어떤 입력에서 어떤 변환을 거쳐 결과가 나왔는지 설명해야 한다. 강사는 변수명, 출력 형태, 저장 여부를 함께 확인하며 다음 실행에서도 같은 결과가 나오는지 묻는다.

### 체크포인트 3: 상태 값 확인한다

코랩과 로컬 실행 차이를 줄이기 위해, 레슨 09 속도 제한, 재시도, 로그에서는 상태 값을/를 확인한다는 과정을 별도의 단계로 둔다. 이때 학생은 화면에 보이는 값 하나보다 어떤 입력에서 어떤 변환을 거쳐 결과가 나왔는지 설명해야 한다. 강사는 변수명, 출력 형태, 저장 여부를 함께 확인하며 다음 실행에서도 같은 결과가 나오는지 묻는다.

### 체크포인트 4: 저장 경로 분리한다

수업 중 피드백 시간을 줄이기 위해, 레슨 09 속도 제한, 재시도, 로그에서는 저장 경로을/를 분리한다는 과정을 별도의 단계로 둔다. 이때 학생은 화면에 보이는 값 하나보다 어떤 입력에서 어떤 변환을 거쳐 결과가 나왔는지 설명해야 한다. 강사는 변수명, 출력 형태, 저장 여부를 함께 확인하며 다음 실행에서도 같은 결과가 나오는지 묻는다.

### 체크포인트 5: 실행 순서 기록한다

학생이 막힌 지점을 찾기 위해, 레슨 09 속도 제한, 재시도, 로그에서는 실행 순서을/를 기록한다는 과정을 별도의 단계로 둔다. 이때 학생은 화면에 보이는 값 하나보다 어떤 입력에서 어떤 변환을 거쳐 결과가 나왔는지 설명해야 한다. 강사는 변수명, 출력 형태, 저장 여부를 함께 확인하며 다음 실행에서도 같은 결과가 나오는지 묻는다.

### 체크포인트 6: 운영 메모 비교한다

운영자가 결과를 이해할 수 있게, 레슨 09 속도 제한, 재시도, 로그에서는 운영 메모을/를 비교한다는 과정을 별도의 단계로 둔다. 이때 학생은 화면에 보이는 값 하나보다 어떤 입력에서 어떤 변환을 거쳐 결과가 나왔는지 설명해야 한다. 강사는 변수명, 출력 형태, 저장 여부를 함께 확인하며 다음 실행에서도 같은 결과가 나오는지 묻는다.

### 체크포인트 7: 출력 형태 요약한다

재실행했을 때 같은 결과를 얻기 위해, 레슨 09 속도 제한, 재시도, 로그에서는 출력 형태을/를 요약한다는 과정을 별도의 단계로 둔다. 이때 학생은 화면에 보이는 값 하나보다 어떤 입력에서 어떤 변환을 거쳐 결과가 나왔는지 설명해야 한다. 강사는 변수명, 출력 형태, 저장 여부를 함께 확인하며 다음 실행에서도 같은 결과가 나오는지 묻는다.

### 체크포인트 8: 반복 단위 검토한다

다음 셀에서 재사용하기 위해, 레슨 09 속도 제한, 재시도, 로그에서는 반복 단위을/를 검토한다는 과정을 별도의 단계로 둔다. 이때 학생은 화면에 보이는 값 하나보다 어떤 입력에서 어떤 변환을 거쳐 결과가 나왔는지 설명해야 한다. 강사는 변수명, 출력 형태, 저장 여부를 함께 확인하며 다음 실행에서도 같은 결과가 나오는지 묻는다.

### 체크포인트 9: 상태 값 정리한다

실제 사이트 확장 전에 위험을 낮추기 위해, 레슨 09 속도 제한, 재시도, 로그에서는 상태 값을/를 정리한다는 과정을 별도의 단계로 둔다. 이때 학생은 화면에 보이는 값 하나보다 어떤 입력에서 어떤 변환을 거쳐 결과가 나왔는지 설명해야 한다. 강사는 변수명, 출력 형태, 저장 여부를 함께 확인하며 다음 실행에서도 같은 결과가 나오는지 묻는다.

### 체크포인트 10: 저장 경로 설명한다

저장 파일의 신뢰도를 높이기 위해, 레슨 09 속도 제한, 재시도, 로그에서는 저장 경로을/를 설명한다는 과정을 별도의 단계로 둔다. 이때 학생은 화면에 보이는 값 하나보다 어떤 입력에서 어떤 변환을 거쳐 결과가 나왔는지 설명해야 한다. 강사는 변수명, 출력 형태, 저장 여부를 함께 확인하며 다음 실행에서도 같은 결과가 나오는지 묻는다.

### 체크포인트 11: 실행 순서 저장한다

코랩과 로컬 실행 차이를 줄이기 위해, 레슨 09 속도 제한, 재시도, 로그에서는 실행 순서을/를 저장한다는 과정을 별도의 단계로 둔다. 이때 학생은 화면에 보이는 값 하나보다 어떤 입력에서 어떤 변환을 거쳐 결과가 나왔는지 설명해야 한다. 강사는 변수명, 출력 형태, 저장 여부를 함께 확인하며 다음 실행에서도 같은 결과가 나오는지 묻는다.

### 체크포인트 12: 운영 메모 되돌아본다

수업 중 피드백 시간을 줄이기 위해, 레슨 09 속도 제한, 재시도, 로그에서는 운영 메모을/를 되돌아본다는 과정을 별도의 단계로 둔다. 이때 학생은 화면에 보이는 값 하나보다 어떤 입력에서 어떤 변환을 거쳐 결과가 나왔는지 설명해야 한다. 강사는 변수명, 출력 형태, 저장 여부를 함께 확인하며 다음 실행에서도 같은 결과가 나오는지 묻는다.

### 체크포인트 13: 출력 형태 표준화한다

학생이 막힌 지점을 찾기 위해, 레슨 09 속도 제한, 재시도, 로그에서는 출력 형태을/를 표준화한다는 과정을 별도의 단계로 둔다. 이때 학생은 화면에 보이는 값 하나보다 어떤 입력에서 어떤 변환을 거쳐 결과가 나왔는지 설명해야 한다. 강사는 변수명, 출력 형태, 저장 여부를 함께 확인하며 다음 실행에서도 같은 결과가 나오는지 묻는다.

### 체크포인트 14: 반복 단위 검증한다

운영자가 결과를 이해할 수 있게, 레슨 09 속도 제한, 재시도, 로그에서는 반복 단위을/를 검증한다는 과정을 별도의 단계로 둔다. 이때 학생은 화면에 보이는 값 하나보다 어떤 입력에서 어떤 변환을 거쳐 결과가 나왔는지 설명해야 한다. 강사는 변수명, 출력 형태, 저장 여부를 함께 확인하며 다음 실행에서도 같은 결과가 나오는지 묻는다.

### 체크포인트 15: 상태 값 확인한다

재실행했을 때 같은 결과를 얻기 위해, 레슨 09 속도 제한, 재시도, 로그에서는 상태 값을/를 확인한다는 과정을 별도의 단계로 둔다. 이때 학생은 화면에 보이는 값 하나보다 어떤 입력에서 어떤 변환을 거쳐 결과가 나왔는지 설명해야 한다. 강사는 변수명, 출력 형태, 저장 여부를 함께 확인하며 다음 실행에서도 같은 결과가 나오는지 묻는다.

### 체크포인트 16: 저장 경로 분리한다

다음 셀에서 재사용하기 위해, 레슨 09 속도 제한, 재시도, 로그에서는 저장 경로을/를 분리한다는 과정을 별도의 단계로 둔다. 이때 학생은 화면에 보이는 값 하나보다 어떤 입력에서 어떤 변환을 거쳐 결과가 나왔는지 설명해야 한다. 강사는 변수명, 출력 형태, 저장 여부를 함께 확인하며 다음 실행에서도 같은 결과가 나오는지 묻는다.

### 체크포인트 17: 실행 순서 기록한다

실제 사이트 확장 전에 위험을 낮추기 위해, 레슨 09 속도 제한, 재시도, 로그에서는 실행 순서을/를 기록한다는 과정을 별도의 단계로 둔다. 이때 학생은 화면에 보이는 값 하나보다 어떤 입력에서 어떤 변환을 거쳐 결과가 나왔는지 설명해야 한다. 강사는 변수명, 출력 형태, 저장 여부를 함께 확인하며 다음 실행에서도 같은 결과가 나오는지 묻는다.

### 체크포인트 18: 운영 메모 비교한다

저장 파일의 신뢰도를 높이기 위해, 레슨 09 속도 제한, 재시도, 로그에서는 운영 메모을/를 비교한다는 과정을 별도의 단계로 둔다. 이때 학생은 화면에 보이는 값 하나보다 어떤 입력에서 어떤 변환을 거쳐 결과가 나왔는지 설명해야 한다. 강사는 변수명, 출력 형태, 저장 여부를 함께 확인하며 다음 실행에서도 같은 결과가 나오는지 묻는다.

### 체크포인트 19: 출력 형태 요약한다

코랩과 로컬 실행 차이를 줄이기 위해, 레슨 09 속도 제한, 재시도, 로그에서는 출력 형태을/를 요약한다는 과정을 별도의 단계로 둔다. 이때 학생은 화면에 보이는 값 하나보다 어떤 입력에서 어떤 변환을 거쳐 결과가 나왔는지 설명해야 한다. 강사는 변수명, 출력 형태, 저장 여부를 함께 확인하며 다음 실행에서도 같은 결과가 나오는지 묻는다.

### 체크포인트 20: 반복 단위 검토한다

수업 중 피드백 시간을 줄이기 위해, 레슨 09 속도 제한, 재시도, 로그에서는 반복 단위을/를 검토한다는 과정을 별도의 단계로 둔다. 이때 학생은 화면에 보이는 값 하나보다 어떤 입력에서 어떤 변환을 거쳐 결과가 나왔는지 설명해야 한다. 강사는 변수명, 출력 형태, 저장 여부를 함께 확인하며 다음 실행에서도 같은 결과가 나오는지 묻는다.

### 체크포인트 21: 상태 값 정리한다

학생이 막힌 지점을 찾기 위해, 레슨 09 속도 제한, 재시도, 로그에서는 상태 값을/를 정리한다는 과정을 별도의 단계로 둔다. 이때 학생은 화면에 보이는 값 하나보다 어떤 입력에서 어떤 변환을 거쳐 결과가 나왔는지 설명해야 한다. 강사는 변수명, 출력 형태, 저장 여부를 함께 확인하며 다음 실행에서도 같은 결과가 나오는지 묻는다.
