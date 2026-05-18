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

# 레슨 07 — 실습 문제 정답지

[![Open in Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/Kevin-innovation/jupyter-lecture/blob/main/python-web-automation/lectures/07/%EB%A0%88%EC%8A%A8%2007%20%E2%80%94%20%EC%95%88%EC%A0%95%EC%A0%81%EC%9D%B8%20selector%EC%99%80%20wait.ipynb)

> 🔒 교사·관리자 전용. 학생에게 배포 금지.

안정적인 selector와 wait 실습 문제의 모범 답안이다. 출력값만 보지 말고 상태 변화, selector 안정성, 로그를 함께 확인한다.

## 0. 환경 셀

~~~python
import os
import re
import time
import csv
from pathlib import Path
from urllib.parse import urljoin, urlparse, parse_qs, urlencode

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
    DATA_BASE = 'https://raw.githubusercontent.com/Kevin-innovation/jupyter-lecture/main/python-web-automation/lectures/07/data'
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

def clean_int(text):
    return int(re.sub(r'[^0-9]', '', str(text)))

class MiniLocator:
    def __init__(self, page, selector):
        self.page = page
        self.selector = selector
    def elements(self):
        return self.page.soup.select(self.selector)
    def count(self):
        return len(self.elements())
    def first(self):
        items = self.elements()
        if not items:
            raise ValueError(f'no element for {self.selector}')
        return items[0]
    def text_content(self):
        return self.first().get_text(' ', strip=True)
    def all_text_contents(self):
        return [el.get_text(' ', strip=True) for el in self.elements()]
    def get_attribute(self, name):
        return self.first().get(name)
    def fill(self, value):
        self.first()['value'] = str(value)
    def click(self):
        return self.page._click(self.first())

class MiniPage:
    def __init__(self, html):
        self.soup = BeautifulSoup(html, 'html.parser')
        self.step = 0
        self.log = []
    def locator(self, selector):
        return MiniLocator(self, selector)
    def text_content(self, selector):
        return self.locator(selector).text_content()
    def fill(self, selector, value):
        self.locator(selector).fill(value)
        self.log.append({'action': 'fill', 'selector': selector, 'value': str(value)})
    def click(self, selector):
        result = self.locator(selector).click()
        self.log.append({'action': 'click', 'selector': selector, 'result': result})
        return result
    def _value(self, selector):
        el = self.soup.select_one(selector)
        return '' if el is None else el.get('value', '')
    def _click(self, el):
        action = el.get('data-action', '')
        if action == 'submit-profile':
            name = self._value('#student-name')
            course = self._value('#course-name')
            memo = self._value('#memo')
            out = self.soup.select_one('#result')
            out.string = f'{name} / {course} / {memo}'
            out['data-state'] = 'submitted'
            return 'submitted'
        if action == 'toggle-complete':
            target = self.soup.select_one(el.get('data-target', ''))
            if target:
                target['data-status'] = 'done' if target.get('data-status') != 'done' else 'pending'
                return target['data-status']
        if action == 'open-tab':
            target_id = el.get('data-target')
            for panel in self.soup.select('[role="tabpanel"]'):
                panel['hidden'] = 'true'
            target = self.soup.select_one(f'#{target_id}')
            if target and target.has_attr('hidden'):
                del target['hidden']
            return target_id
        return action or 'clicked'
    def visible_elements(self, selector):
        items = []
        for el in self.soup.select(selector):
            delay = int(el.get('data-delay-step', '0'))
            hidden = el.has_attr('hidden') or el.get('aria-hidden') == 'true'
            if delay <= self.step and not hidden:
                items.append(el)
        return items
    def tick(self):
        self.step += 1
        return self.step
    def wait_for_selector(self, selector, timeout_steps=5):
        for _ in range(timeout_steps + 1):
            items = self.visible_elements(selector)
            if items:
                return items[0]
            self.tick()
        raise TimeoutError(f'timeout waiting for {selector}')

print('colab:', IS_COLAB)
print('data base:', DATA_BASE)
~~~

---

## 문제 1 정답 — 안정 selector로 제목 읽기

~~~python
page = MiniPage(load_text('dynamic_dashboard.html'))
print(page.text_content('[data-testid="page-title"]'))
~~~

### 왜 이 코드가 정답인지

이 답안은 문제에서 요구한 상태나 화면 구조를 먼저 명확한 자료구조로 바꾼 뒤 필요한 값만 선택한다. data-testid는 화면 문구나 임의 class보다 안정적인 기준이다. 결과가 맞아도 세션 상태, locator, wait, 로그가 코드에 남아 있지 않으면 운영형 자동화로 보기 어렵다.

### 채점 포인트

- 입력 fixture를 환경 셀의 헬퍼로 읽어 코랩과 로컬에서 모두 동작하는가.
- 상태 변화, 버튼 클릭, wait 조건을 중간 변수나 로그로 확인할 수 있는가.
- selector가 화면 문구보다 id, data-testid, role 같은 안정적인 기준을 우선하는가.
- 출력 형태가 문제 요구사항과 일치하고 CSV 저장 시 헤더가 있는가.

### 자주 보이는 오답

- 화면 텍스트만 믿고 불안정한 selector를 사용한다.
- 상태가 바뀌기 전에 결과를 읽어 빈 값이나 이전 값이 나온다.
- 쿠키나 폼 입력 값을 문자열 변수에만 두고 세션/페이지 상태에 반영하지 않는다.
- 최종 미션에서 개별 문제 코드를 복사만 해서 함수화와 로그가 빠진다.

---

## 문제 2 정답 — 불안정 class와 안정 selector 비교

~~~python
lab = MiniPage(load_text('selector_lab.html'))
print(lab.locator('.card').count())
print(lab.locator('[data-testid="lesson-card"]').count())
~~~

### 왜 이 코드가 정답인지

이 답안은 문제에서 요구한 상태나 화면 구조를 먼저 명확한 자료구조로 바꾼 뒤 필요한 값만 선택한다. class는 스타일 변경에 취약하고 data-testid는 테스트 목적이 분명하다. 결과가 맞아도 세션 상태, locator, wait, 로그가 코드에 남아 있지 않으면 운영형 자동화로 보기 어렵다.

### 채점 포인트

- 입력 fixture를 환경 셀의 헬퍼로 읽어 코랩과 로컬에서 모두 동작하는가.
- 상태 변화, 버튼 클릭, wait 조건을 중간 변수나 로그로 확인할 수 있는가.
- selector가 화면 문구보다 id, data-testid, role 같은 안정적인 기준을 우선하는가.
- 출력 형태가 문제 요구사항과 일치하고 CSV 저장 시 헤더가 있는가.

### 자주 보이는 오답

- 화면 텍스트만 믿고 불안정한 selector를 사용한다.
- 상태가 바뀌기 전에 결과를 읽어 빈 값이나 이전 값이 나온다.
- 쿠키나 폼 입력 값을 문자열 변수에만 두고 세션/페이지 상태에 반영하지 않는다.
- 최종 미션에서 개별 문제 코드를 복사만 해서 함수화와 로그가 빠진다.

---

## 문제 3 정답 — 저장 버튼 selector 선택

~~~python
print(lab.locator('[data-testid="save-button"]').text_content())
~~~

### 왜 이 코드가 정답인지

이 답안은 문제에서 요구한 상태나 화면 구조를 먼저 명확한 자료구조로 바꾼 뒤 필요한 값만 선택한다. 버튼 문구보다 data-testid가 유지보수에 적합하다. 결과가 맞아도 세션 상태, locator, wait, 로그가 코드에 남아 있지 않으면 운영형 자동화로 보기 어렵다.

### 채점 포인트

- 입력 fixture를 환경 셀의 헬퍼로 읽어 코랩과 로컬에서 모두 동작하는가.
- 상태 변화, 버튼 클릭, wait 조건을 중간 변수나 로그로 확인할 수 있는가.
- selector가 화면 문구보다 id, data-testid, role 같은 안정적인 기준을 우선하는가.
- 출력 형태가 문제 요구사항과 일치하고 CSV 저장 시 헤더가 있는가.

### 자주 보이는 오답

- 화면 텍스트만 믿고 불안정한 selector를 사용한다.
- 상태가 바뀌기 전에 결과를 읽어 빈 값이나 이전 값이 나온다.
- 쿠키나 폼 입력 값을 문자열 변수에만 두고 세션/페이지 상태에 반영하지 않는다.
- 최종 미션에서 개별 문제 코드를 복사만 해서 함수화와 로그가 빠진다.

---

## 문제 4 정답 — 대시보드 즉시 보이는 metric

~~~python
print(page.step)
print(page.wait_for_selector('[data-testid="metric-active"]', timeout_steps=0).get_text(' ', strip=True))
~~~

### 왜 이 코드가 정답인지

이 답안은 문제에서 요구한 상태나 화면 구조를 먼저 명확한 자료구조로 바꾼 뒤 필요한 값만 선택한다. delay 0 요소는 wait 없이도 바로 보인다. 결과가 맞아도 세션 상태, locator, wait, 로그가 코드에 남아 있지 않으면 운영형 자동화로 보기 어렵다.

### 채점 포인트

- 입력 fixture를 환경 셀의 헬퍼로 읽어 코랩과 로컬에서 모두 동작하는가.
- 상태 변화, 버튼 클릭, wait 조건을 중간 변수나 로그로 확인할 수 있는가.
- selector가 화면 문구보다 id, data-testid, role 같은 안정적인 기준을 우선하는가.
- 출력 형태가 문제 요구사항과 일치하고 CSV 저장 시 헤더가 있는가.

### 자주 보이는 오답

- 화면 텍스트만 믿고 불안정한 selector를 사용한다.
- 상태가 바뀌기 전에 결과를 읽어 빈 값이나 이전 값이 나온다.
- 쿠키나 폼 입력 값을 문자열 변수에만 두고 세션/페이지 상태에 반영하지 않는다.
- 최종 미션에서 개별 문제 코드를 복사만 해서 함수화와 로그가 빠진다.

---

## 문제 5 정답 — 한 단계 뒤 나타나는 metric 기다리기

~~~python
el = page.wait_for_selector('[data-testid="metric-submit"]', timeout_steps=2)
print(page.step)
print(el.get_text(' ', strip=True))
~~~

### 왜 이 코드가 정답인지

이 답안은 문제에서 요구한 상태나 화면 구조를 먼저 명확한 자료구조로 바꾼 뒤 필요한 값만 선택한다. wait는 조건이 충족될 때까지 확인 단계를 늘린다. 결과가 맞아도 세션 상태, locator, wait, 로그가 코드에 남아 있지 않으면 운영형 자동화로 보기 어렵다.

### 채점 포인트

- 입력 fixture를 환경 셀의 헬퍼로 읽어 코랩과 로컬에서 모두 동작하는가.
- 상태 변화, 버튼 클릭, wait 조건을 중간 변수나 로그로 확인할 수 있는가.
- selector가 화면 문구보다 id, data-testid, role 같은 안정적인 기준을 우선하는가.
- 출력 형태가 문제 요구사항과 일치하고 CSV 저장 시 헤더가 있는가.

### 자주 보이는 오답

- 화면 텍스트만 믿고 불안정한 selector를 사용한다.
- 상태가 바뀌기 전에 결과를 읽어 빈 값이나 이전 값이 나온다.
- 쿠키나 폼 입력 값을 문자열 변수에만 두고 세션/페이지 상태에 반영하지 않는다.
- 최종 미션에서 개별 문제 코드를 복사만 해서 함수화와 로그가 빠진다.

---

## 문제 6 정답 — 두 단계 뒤 나타나는 metric 기다리기

~~~python
el = page.wait_for_selector('[data-testid="metric-pass"]', timeout_steps=3)
print(page.step)
print(el.get_text(' ', strip=True))
~~~

### 왜 이 코드가 정답인지

이 답안은 문제에서 요구한 상태나 화면 구조를 먼저 명확한 자료구조로 바꾼 뒤 필요한 값만 선택한다. 느리게 표시되는 요소도 충분한 timeout이면 안정적으로 읽을 수 있다. 결과가 맞아도 세션 상태, locator, wait, 로그가 코드에 남아 있지 않으면 운영형 자동화로 보기 어렵다.

### 채점 포인트

- 입력 fixture를 환경 셀의 헬퍼로 읽어 코랩과 로컬에서 모두 동작하는가.
- 상태 변화, 버튼 클릭, wait 조건을 중간 변수나 로그로 확인할 수 있는가.
- selector가 화면 문구보다 id, data-testid, role 같은 안정적인 기준을 우선하는가.
- 출력 형태가 문제 요구사항과 일치하고 CSV 저장 시 헤더가 있는가.

### 자주 보이는 오답

- 화면 텍스트만 믿고 불안정한 selector를 사용한다.
- 상태가 바뀌기 전에 결과를 읽어 빈 값이나 이전 값이 나온다.
- 쿠키나 폼 입력 값을 문자열 변수에만 두고 세션/페이지 상태에 반영하지 않는다.
- 최종 미션에서 개별 문제 코드를 복사만 해서 함수화와 로그가 빠진다.

---

## 문제 7 정답 — 학생 row 전체 대기 후 개수 세기

~~~python
page.step = 0
page.wait_for_selector('[data-testid="student-row"]', timeout_steps=2)
visible = page.visible_elements('[data-testid="student-row"]')
print(len(visible))
~~~

### 왜 이 코드가 정답인지

이 답안은 문제에서 요구한 상태나 화면 구조를 먼저 명확한 자료구조로 바꾼 뒤 필요한 값만 선택한다. visible_elements는 현재 step에서 보이는 요소만 반환한다. 결과가 맞아도 세션 상태, locator, wait, 로그가 코드에 남아 있지 않으면 운영형 자동화로 보기 어렵다.

### 채점 포인트

- 입력 fixture를 환경 셀의 헬퍼로 읽어 코랩과 로컬에서 모두 동작하는가.
- 상태 변화, 버튼 클릭, wait 조건을 중간 변수나 로그로 확인할 수 있는가.
- selector가 화면 문구보다 id, data-testid, role 같은 안정적인 기준을 우선하는가.
- 출력 형태가 문제 요구사항과 일치하고 CSV 저장 시 헤더가 있는가.

### 자주 보이는 오답

- 화면 텍스트만 믿고 불안정한 selector를 사용한다.
- 상태가 바뀌기 전에 결과를 읽어 빈 값이나 이전 값이 나온다.
- 쿠키나 폼 입력 값을 문자열 변수에만 두고 세션/페이지 상태에 반영하지 않는다.
- 최종 미션에서 개별 문제 코드를 복사만 해서 함수화와 로그가 빠진다.

---

## 문제 8 정답 — Timeout 오류 확인

~~~python
try:
    page.wait_for_selector('[data-testid="missing"]', timeout_steps=1)
except Exception as error:
    print(type(error).__name__)
~~~

### 왜 이 코드가 정답인지

이 답안은 문제에서 요구한 상태나 화면 구조를 먼저 명확한 자료구조로 바꾼 뒤 필요한 값만 선택한다. 없는 selector는 명확히 실패해야 잘못된 자동화를 빨리 찾는다. 결과가 맞아도 세션 상태, locator, wait, 로그가 코드에 남아 있지 않으면 운영형 자동화로 보기 어렵다.

### 채점 포인트

- 입력 fixture를 환경 셀의 헬퍼로 읽어 코랩과 로컬에서 모두 동작하는가.
- 상태 변화, 버튼 클릭, wait 조건을 중간 변수나 로그로 확인할 수 있는가.
- selector가 화면 문구보다 id, data-testid, role 같은 안정적인 기준을 우선하는가.
- 출력 형태가 문제 요구사항과 일치하고 CSV 저장 시 헤더가 있는가.

### 자주 보이는 오답

- 화면 텍스트만 믿고 불안정한 selector를 사용한다.
- 상태가 바뀌기 전에 결과를 읽어 빈 값이나 이전 값이 나온다.
- 쿠키나 폼 입력 값을 문자열 변수에만 두고 세션/페이지 상태에 반영하지 않는다.
- 최종 미션에서 개별 문제 코드를 복사만 해서 함수화와 로그가 빠진다.

---

## 문제 9 정답 — wait_cases CSV 읽기

~~~python
cases = list(csv.DictReader(load_text('wait_cases.csv').splitlines()))
print(len(cases))
print(cases[0]['selector'])
~~~

### 왜 이 코드가 정답인지

이 답안은 문제에서 요구한 상태나 화면 구조를 먼저 명확한 자료구조로 바꾼 뒤 필요한 값만 선택한다. wait 검증도 데이터 케이스로 관리하면 반복 실행이 쉽다. 결과가 맞아도 세션 상태, locator, wait, 로그가 코드에 남아 있지 않으면 운영형 자동화로 보기 어렵다.

### 채점 포인트

- 입력 fixture를 환경 셀의 헬퍼로 읽어 코랩과 로컬에서 모두 동작하는가.
- 상태 변화, 버튼 클릭, wait 조건을 중간 변수나 로그로 확인할 수 있는가.
- selector가 화면 문구보다 id, data-testid, role 같은 안정적인 기준을 우선하는가.
- 출력 형태가 문제 요구사항과 일치하고 CSV 저장 시 헤더가 있는가.

### 자주 보이는 오답

- 화면 텍스트만 믿고 불안정한 selector를 사용한다.
- 상태가 바뀌기 전에 결과를 읽어 빈 값이나 이전 값이 나온다.
- 쿠키나 폼 입력 값을 문자열 변수에만 두고 세션/페이지 상태에 반영하지 않는다.
- 최종 미션에서 개별 문제 코드를 복사만 해서 함수화와 로그가 빠진다.

---

## 문제 10 정답 — wait 케이스 실행하기

~~~python
results = []
for row in cases:
    p = MiniPage(load_text('dynamic_dashboard.html'))
    try:
        el = p.wait_for_selector(row['selector'], timeout_steps=int(row['timeout_steps']))
        text = el.get_text(' ', strip=True)
        results.append({'selector': row['selector'], 'ok': row['expected_text'] in text})
    except Exception:
        results.append({'selector': row['selector'], 'ok': row['expected_text'] == ''})
print(results)
~~~

### 왜 이 코드가 정답인지

이 답안은 문제에서 요구한 상태나 화면 구조를 먼저 명확한 자료구조로 바꾼 뒤 필요한 값만 선택한다. 성공과 실패 케이스를 모두 기록해야 wait 정책을 검증할 수 있다. 결과가 맞아도 세션 상태, locator, wait, 로그가 코드에 남아 있지 않으면 운영형 자동화로 보기 어렵다.

### 채점 포인트

- 입력 fixture를 환경 셀의 헬퍼로 읽어 코랩과 로컬에서 모두 동작하는가.
- 상태 변화, 버튼 클릭, wait 조건을 중간 변수나 로그로 확인할 수 있는가.
- selector가 화면 문구보다 id, data-testid, role 같은 안정적인 기준을 우선하는가.
- 출력 형태가 문제 요구사항과 일치하고 CSV 저장 시 헤더가 있는가.

### 자주 보이는 오답

- 화면 텍스트만 믿고 불안정한 selector를 사용한다.
- 상태가 바뀌기 전에 결과를 읽어 빈 값이나 이전 값이 나온다.
- 쿠키나 폼 입력 값을 문자열 변수에만 두고 세션/페이지 상태에 반영하지 않는다.
- 최종 미션에서 개별 문제 코드를 복사만 해서 함수화와 로그가 빠진다.

---

## 문제 11 정답 — 통과한 wait 케이스 수 세기

~~~python
passed = sum(row['ok'] for row in results)
print(passed, '/', len(results))
~~~

### 왜 이 코드가 정답인지

이 답안은 문제에서 요구한 상태나 화면 구조를 먼저 명확한 자료구조로 바꾼 뒤 필요한 값만 선택한다. 테스트 결과는 개별 로그와 전체 통과 수를 함께 봐야 한다. 결과가 맞아도 세션 상태, locator, wait, 로그가 코드에 남아 있지 않으면 운영형 자동화로 보기 어렵다.

### 채점 포인트

- 입력 fixture를 환경 셀의 헬퍼로 읽어 코랩과 로컬에서 모두 동작하는가.
- 상태 변화, 버튼 클릭, wait 조건을 중간 변수나 로그로 확인할 수 있는가.
- selector가 화면 문구보다 id, data-testid, role 같은 안정적인 기준을 우선하는가.
- 출력 형태가 문제 요구사항과 일치하고 CSV 저장 시 헤더가 있는가.

### 자주 보이는 오답

- 화면 텍스트만 믿고 불안정한 selector를 사용한다.
- 상태가 바뀌기 전에 결과를 읽어 빈 값이나 이전 값이 나온다.
- 쿠키나 폼 입력 값을 문자열 변수에만 두고 세션/페이지 상태에 반영하지 않는다.
- 최종 미션에서 개별 문제 코드를 복사만 해서 함수화와 로그가 빠진다.

---

## 문제 12 정답 — lesson-card id 목록 추출

~~~python
ids = [el['data-lesson-id'] for el in lab.locator('[data-testid="lesson-card"]').elements()]
print(ids)
~~~

### 왜 이 코드가 정답인지

이 답안은 문제에서 요구한 상태나 화면 구조를 먼저 명확한 자료구조로 바꾼 뒤 필요한 값만 선택한다. 카드는 텍스트보다 data-lesson-id 같은 식별자를 저장하는 편이 안정적이다. 결과가 맞아도 세션 상태, locator, wait, 로그가 코드에 남아 있지 않으면 운영형 자동화로 보기 어렵다.

### 채점 포인트

- 입력 fixture를 환경 셀의 헬퍼로 읽어 코랩과 로컬에서 모두 동작하는가.
- 상태 변화, 버튼 클릭, wait 조건을 중간 변수나 로그로 확인할 수 있는가.
- selector가 화면 문구보다 id, data-testid, role 같은 안정적인 기준을 우선하는가.
- 출력 형태가 문제 요구사항과 일치하고 CSV 저장 시 헤더가 있는가.

### 자주 보이는 오답

- 화면 텍스트만 믿고 불안정한 selector를 사용한다.
- 상태가 바뀌기 전에 결과를 읽어 빈 값이나 이전 값이 나온다.
- 쿠키나 폼 입력 값을 문자열 변수에만 두고 세션/페이지 상태에 반영하지 않는다.
- 최종 미션에서 개별 문제 코드를 복사만 해서 함수화와 로그가 빠진다.

---

## 문제 13 정답 — selector 추천표 만들기

~~~python
selector_rows = []
selector_rows.append({'purpose': 'save', 'selector': '[data-testid="save-button"]'})
selector_rows.append({'purpose': 'card', 'selector': '[data-testid="lesson-card"]'})
print(selector_rows)
~~~

### 왜 이 코드가 정답인지

이 답안은 문제에서 요구한 상태나 화면 구조를 먼저 명확한 자료구조로 바꾼 뒤 필요한 값만 선택한다. 좋은 자동화는 selector 선택 이유를 기록한다. 결과가 맞아도 세션 상태, locator, wait, 로그가 코드에 남아 있지 않으면 운영형 자동화로 보기 어렵다.

### 채점 포인트

- 입력 fixture를 환경 셀의 헬퍼로 읽어 코랩과 로컬에서 모두 동작하는가.
- 상태 변화, 버튼 클릭, wait 조건을 중간 변수나 로그로 확인할 수 있는가.
- selector가 화면 문구보다 id, data-testid, role 같은 안정적인 기준을 우선하는가.
- 출력 형태가 문제 요구사항과 일치하고 CSV 저장 시 헤더가 있는가.

### 자주 보이는 오답

- 화면 텍스트만 믿고 불안정한 selector를 사용한다.
- 상태가 바뀌기 전에 결과를 읽어 빈 값이나 이전 값이 나온다.
- 쿠키나 폼 입력 값을 문자열 변수에만 두고 세션/페이지 상태에 반영하지 않는다.
- 최종 미션에서 개별 문제 코드를 복사만 해서 함수화와 로그가 빠진다.

---

## 문제 14 정답 — selector 결과 CSV 저장

~~~python
with open('lesson07_wait_results.csv', 'w', newline='', encoding='utf-8') as f:
    writer = csv.DictWriter(f, fieldnames=['selector', 'ok'])
    writer.writeheader()
    writer.writerows(results)
print('saved:', len(results))
~~~

### 왜 이 코드가 정답인지

이 답안은 문제에서 요구한 상태나 화면 구조를 먼저 명확한 자료구조로 바꾼 뒤 필요한 값만 선택한다. wait 결과를 CSV로 남겨야 어떤 selector가 흔들리는지 추적할 수 있다. 결과가 맞아도 세션 상태, locator, wait, 로그가 코드에 남아 있지 않으면 운영형 자동화로 보기 어렵다.

### 채점 포인트

- 입력 fixture를 환경 셀의 헬퍼로 읽어 코랩과 로컬에서 모두 동작하는가.
- 상태 변화, 버튼 클릭, wait 조건을 중간 변수나 로그로 확인할 수 있는가.
- selector가 화면 문구보다 id, data-testid, role 같은 안정적인 기준을 우선하는가.
- 출력 형태가 문제 요구사항과 일치하고 CSV 저장 시 헤더가 있는가.

### 자주 보이는 오답

- 화면 텍스트만 믿고 불안정한 selector를 사용한다.
- 상태가 바뀌기 전에 결과를 읽어 빈 값이나 이전 값이 나온다.
- 쿠키나 폼 입력 값을 문자열 변수에만 두고 세션/페이지 상태에 반영하지 않는다.
- 최종 미션에서 개별 문제 코드를 복사만 해서 함수화와 로그가 빠진다.

---

## 문제 15 정답 — 안정화 요약 문장 만들기

~~~python
print(f'wait 케이스 {passed}/{len(results)} 통과, 추천 selector {len(selector_rows)}개를 기록했습니다.')
~~~

### 왜 이 코드가 정답인지

이 답안은 문제에서 요구한 상태나 화면 구조를 먼저 명확한 자료구조로 바꾼 뒤 필요한 값만 선택한다. 마지막에는 숫자 근거가 들어간 운영 요약 문장으로 끝낸다. 결과가 맞아도 세션 상태, locator, wait, 로그가 코드에 남아 있지 않으면 운영형 자동화로 보기 어렵다.

### 채점 포인트

- 입력 fixture를 환경 셀의 헬퍼로 읽어 코랩과 로컬에서 모두 동작하는가.
- 상태 변화, 버튼 클릭, wait 조건을 중간 변수나 로그로 확인할 수 있는가.
- selector가 화면 문구보다 id, data-testid, role 같은 안정적인 기준을 우선하는가.
- 출력 형태가 문제 요구사항과 일치하고 CSV 저장 시 헤더가 있는가.

### 자주 보이는 오답

- 화면 텍스트만 믿고 불안정한 selector를 사용한다.
- 상태가 바뀌기 전에 결과를 읽어 빈 값이나 이전 값이 나온다.
- 쿠키나 폼 입력 값을 문자열 변수에만 두고 세션/페이지 상태에 반영하지 않는다.
- 최종 미션에서 개별 문제 코드를 복사만 해서 함수화와 로그가 빠진다.

---
