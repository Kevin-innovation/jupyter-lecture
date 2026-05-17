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

# 레슨 07 — 실습 문제

안정적인 selector와 wait 레슨의 학생용 문제 노트북이다. 강의 노트북을 먼저 실행한 뒤 빈칸을 직접 채운다.

## 통과 기준

- 총 15문제 중 12문제 이상 정상 출력이면 통과.
- 문제 1~5는 구조 확인, 6~10은 상태 변화와 반복 처리, 11~15는 로그와 저장이다.
- 정답값은 적지 않는다. 출력 형태와 fixture 구조를 보고 직접 판단한다.

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

## 문제 1 — 안정 selector로 제목 읽기

지시된 값을 코드로 추출하거나 상태를 변경한다.

**기대 결과 형태**: 요구한 값이 한 줄 또는 리스트/딕셔너리 형태로 출력된다.

**빈칸 힌트**: HTML 구조, CSV 헤더, 이전 예제를 확인한 뒤 `____` 부분을 직접 채운다.

~~~python
page = MiniPage(load_text('____'))
print(page.text_content('____'))
~~~

---

## 문제 2 — 불안정 class와 안정 selector 비교

지시된 값을 코드로 추출하거나 상태를 변경한다.

**기대 결과 형태**: 요구한 값이 한 줄 또는 리스트/딕셔너리 형태로 출력된다.

**빈칸 힌트**: HTML 구조, CSV 헤더, 이전 예제를 확인한 뒤 `____` 부분을 직접 채운다.

~~~python
lab = MiniPage(load_text('____'))
print(lab.locator('____').count())
print(lab.locator('____').count())
~~~

---

## 문제 3 — 저장 버튼 selector 선택

지시된 값을 코드로 추출하거나 상태를 변경한다.

**기대 결과 형태**: 요구한 값이 한 줄 또는 리스트/딕셔너리 형태로 출력된다.

**빈칸 힌트**: HTML 구조, CSV 헤더, 이전 예제를 확인한 뒤 `____` 부분을 직접 채운다.

~~~python
print(lab.locator('____').text_content())
~~~

---

## 문제 4 — 대시보드 즉시 보이는 metric

지시된 값을 코드로 추출하거나 상태를 변경한다.

**기대 결과 형태**: 요구한 값이 한 줄 또는 리스트/딕셔너리 형태로 출력된다.

**빈칸 힌트**: HTML 구조, CSV 헤더, 이전 예제를 확인한 뒤 `____` 부분을 직접 채운다.

~~~python
print(page.step)
print(page.wait_for_selector('____', timeout_steps=0).get_text(' ', strip=True))
~~~

---

## 문제 5 — 한 단계 뒤 나타나는 metric 기다리기

지시된 값을 코드로 추출하거나 상태를 변경한다.

**기대 결과 형태**: 요구한 값이 한 줄 또는 리스트/딕셔너리 형태로 출력된다.

**빈칸 힌트**: HTML 구조, CSV 헤더, 이전 예제를 확인한 뒤 `____` 부분을 직접 채운다.

~~~python
el = page.wait_for_selector('____', timeout_steps=____)
print(page.step)
print(el.get_text(' ', strip=True))
~~~

---

## 문제 6 — 두 단계 뒤 나타나는 metric 기다리기

지시된 값을 코드로 추출하거나 상태를 변경한다.

**기대 결과 형태**: 요구한 값이 한 줄 또는 리스트/딕셔너리 형태로 출력된다.

**빈칸 힌트**: HTML 구조, CSV 헤더, 이전 예제를 확인한 뒤 `____` 부분을 직접 채운다.

~~~python
el = page.wait_for_selector('____', timeout_steps=____)
print(page.step)
print(el.get_text(' ', strip=True))
~~~

---

## 문제 7 — 학생 row 전체 대기 후 개수 세기

지시된 값을 코드로 추출하거나 상태를 변경한다.

**기대 결과 형태**: 요구한 값이 한 줄 또는 리스트/딕셔너리 형태로 출력된다.

**빈칸 힌트**: HTML 구조, CSV 헤더, 이전 예제를 확인한 뒤 `____` 부분을 직접 채운다.

~~~python
page.step = 0
page.wait_for_selector('____', timeout_steps=2)
visible = page.visible_elements('____')
print(len(visible))
~~~

---

## 문제 8 — Timeout 오류 확인

지시된 값을 코드로 추출하거나 상태를 변경한다.

**기대 결과 형태**: 요구한 값이 한 줄 또는 리스트/딕셔너리 형태로 출력된다.

**빈칸 힌트**: HTML 구조, CSV 헤더, 이전 예제를 확인한 뒤 `____` 부분을 직접 채운다.

~~~python
try:
    page.wait_for_selector('____', timeout_steps=1)
except Exception as error:
    print(type(error).__name__)
~~~

---

## 문제 9 — wait_cases CSV 읽기

지시된 값을 코드로 추출하거나 상태를 변경한다.

**기대 결과 형태**: 요구한 값이 한 줄 또는 리스트/딕셔너리 형태로 출력된다.

**빈칸 힌트**: HTML 구조, CSV 헤더, 이전 예제를 확인한 뒤 `____` 부분을 직접 채운다.

~~~python
cases = list(csv.DictReader(load_text('____').splitlines()))
print(len(cases))
print(cases[0]['____'])
~~~

---

## 문제 10 — wait 케이스 실행하기

지시된 값을 코드로 추출하거나 상태를 변경한다.

**기대 결과 형태**: 요구한 값이 한 줄 또는 리스트/딕셔너리 형태로 출력된다.

**빈칸 힌트**: HTML 구조, CSV 헤더, 이전 예제를 확인한 뒤 `____` 부분을 직접 채운다.

~~~python
results = []
for row in cases:
    p = MiniPage(load_text('dynamic_dashboard.html'))
    try:
        el = p.wait_for_selector(row['____'], timeout_steps=int(row['____']))
        text = el.get_text(' ', strip=True)
        results.append({'selector': row['selector'], 'ok': row['expected_text'] in text})
    except Exception:
        results.append({'selector': row['selector'], 'ok': row['expected_text'] == ''})
print(results)
~~~

---

## 문제 11 — 통과한 wait 케이스 수 세기

지시된 값을 코드로 추출하거나 상태를 변경한다.

**기대 결과 형태**: 요구한 값이 한 줄 또는 리스트/딕셔너리 형태로 출력된다.

**빈칸 힌트**: HTML 구조, CSV 헤더, 이전 예제를 확인한 뒤 `____` 부분을 직접 채운다.

~~~python
passed = sum(row['____'] for row in results)
print(passed, '/', len(results))
~~~

---

## 문제 12 — lesson-card id 목록 추출

지시된 값을 코드로 추출하거나 상태를 변경한다.

**기대 결과 형태**: 요구한 값이 한 줄 또는 리스트/딕셔너리 형태로 출력된다.

**빈칸 힌트**: HTML 구조, CSV 헤더, 이전 예제를 확인한 뒤 `____` 부분을 직접 채운다.

~~~python
ids = [el['____'] for el in lab.locator('[data-testid="lesson-card"]').elements()]
print(ids)
~~~

---

## 문제 13 — selector 추천표 만들기

지시된 값을 코드로 추출하거나 상태를 변경한다.

**기대 결과 형태**: 요구한 값이 한 줄 또는 리스트/딕셔너리 형태로 출력된다.

**빈칸 힌트**: HTML 구조, CSV 헤더, 이전 예제를 확인한 뒤 `____` 부분을 직접 채운다.

~~~python
selector_rows = []
selector_rows.append({'purpose': 'save', 'selector': '____'})
selector_rows.append({'purpose': 'card', 'selector': '____'})
print(selector_rows)
~~~

---

## 문제 14 — selector 결과 CSV 저장

지시된 값을 코드로 추출하거나 상태를 변경한다.

**기대 결과 형태**: 요구한 값이 한 줄 또는 리스트/딕셔너리 형태로 출력된다.

**빈칸 힌트**: HTML 구조, CSV 헤더, 이전 예제를 확인한 뒤 `____` 부분을 직접 채운다.

~~~python
with open('lesson07_wait_results.csv', 'w', newline='', encoding='utf-8') as f:
    writer = csv.DictWriter(f, fieldnames=['selector', 'ok'])
    writer.____()
    writer.____(results)
print('saved:', len(results))
~~~

---

## 문제 15 — 안정화 요약 문장 만들기

지시된 값을 코드로 추출하거나 상태를 변경한다.

**기대 결과 형태**: 요구한 값이 한 줄 또는 리스트/딕셔너리 형태로 출력된다.

**빈칸 힌트**: HTML 구조, CSV 헤더, 이전 예제를 확인한 뒤 `____` 부분을 직접 채운다.

~~~python
print(f'wait 케이스 {____}/{len(results)} 통과, 추천 selector {len(____)}개를 기록했습니다.')
~~~

---

### 보강 설명 1

레슨 07 실습 문제은 실행 결과만 맞추는 것이 아니라 재현 가능한 절차를 남기는 것이 핵심이다. 입력 파일, 반복 단위, selector 또는 상태 변수, 저장 경로를 분리하면 오류 위치를 빠르게 찾을 수 있다.

### 보강 설명 2

학생이 막히면 완성 코드를 보여주기보다 HTML 구조, CSV 헤더, 상태 변화 순서를 먼저 말로 설명하게 한다. 구조를 설명할 수 있으면 코드도 안정된다.

### 보강 설명 3

실제 자동화는 다음 주에도 다시 실행되어야 한다. 그래서 쿠키, 폼 입력, selector, wait, 로그 같은 운영 요소를 초반부터 명시적으로 다룬다.

### 보강 설명 4

외부 사이트로 확장할 때는 약관, robots.txt, 요청 간격, 개인정보 여부를 먼저 확인한다. 이 코스의 fixture는 안전한 반복 연습을 위한 합성 데이터다.

### 보강 설명 5

레슨 07 실습 문제은 실행 결과만 맞추는 것이 아니라 재현 가능한 절차를 남기는 것이 핵심이다. 입력 파일, 반복 단위, selector 또는 상태 변수, 저장 경로를 분리하면 오류 위치를 빠르게 찾을 수 있다.

### 보강 설명 6

학생이 막히면 완성 코드를 보여주기보다 HTML 구조, CSV 헤더, 상태 변화 순서를 먼저 말로 설명하게 한다. 구조를 설명할 수 있으면 코드도 안정된다.

### 보강 설명 7

실제 자동화는 다음 주에도 다시 실행되어야 한다. 그래서 쿠키, 폼 입력, selector, wait, 로그 같은 운영 요소를 초반부터 명시적으로 다룬다.

### 보강 설명 8

외부 사이트로 확장할 때는 약관, robots.txt, 요청 간격, 개인정보 여부를 먼저 확인한다. 이 코스의 fixture는 안전한 반복 연습을 위한 합성 데이터다.

### 보강 설명 9

레슨 07 실습 문제은 실행 결과만 맞추는 것이 아니라 재현 가능한 절차를 남기는 것이 핵심이다. 입력 파일, 반복 단위, selector 또는 상태 변수, 저장 경로를 분리하면 오류 위치를 빠르게 찾을 수 있다.
