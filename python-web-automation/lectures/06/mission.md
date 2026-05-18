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

# 레슨 06 — 실습 문제

[![Open in Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/Kevin-innovation/jupyter-lecture/blob/main/python-web-automation/lectures/06/%5B%ED%95%99%EC%83%9D%EC%9A%A9%5D%20%EB%A0%88%EC%8A%A8%2006%20%E2%80%94%20%EB%B8%8C%EB%9D%BC%EC%9A%B0%EC%A0%80%20%EC%9E%90%EB%8F%99%ED%99%94%20%EC%9E%85%EB%AC%B8.ipynb)

브라우저 자동화 입문 레슨의 학생용 문제 노트북이다. 강의 노트북을 먼저 실행한 뒤 빈칸을 직접 채운다.

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
    DATA_BASE = 'https://raw.githubusercontent.com/Kevin-innovation/jupyter-lecture/main/python-web-automation/lectures/06/data'
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

## 문제 1 — 폼 페이지 제목 읽기

지시된 값을 코드로 추출하거나 상태를 변경한다.

**기대 결과 형태**: 요구한 값이 한 줄 또는 리스트/딕셔너리 형태로 출력된다.

**빈칸 힌트**: HTML 구조, CSV 헤더, 이전 예제를 확인한 뒤 `____` 부분을 직접 채운다.

~~~python
page = MiniPage(load_text('____'))
print(page.text_content('____'))
~~~

---

## 문제 2 — input 개수 세기

지시된 값을 코드로 추출하거나 상태를 변경한다.

**기대 결과 형태**: 요구한 값이 한 줄 또는 리스트/딕셔너리 형태로 출력된다.

**빈칸 힌트**: HTML 구조, CSV 헤더, 이전 예제를 확인한 뒤 `____` 부분을 직접 채운다.

~~~python
print(page.locator('____').count())
~~~

---

## 문제 3 — 이름 입력하기

지시된 값을 코드로 추출하거나 상태를 변경한다.

**기대 결과 형태**: 요구한 값이 한 줄 또는 리스트/딕셔너리 형태로 출력된다.

**빈칸 힌트**: HTML 구조, CSV 헤더, 이전 예제를 확인한 뒤 `____` 부분을 직접 채운다.

~~~python
page.fill('____', '김도윤')
print(page.locator('____').get_attribute('____'))
~~~

---

## 문제 4 — 과정과 메모 입력하기

지시된 값을 코드로 추출하거나 상태를 변경한다.

**기대 결과 형태**: 요구한 값이 한 줄 또는 리스트/딕셔너리 형태로 출력된다.

**빈칸 힌트**: HTML 구조, CSV 헤더, 이전 예제를 확인한 뒤 `____` 부분을 직접 채운다.

~~~python
page.fill('____', 'Python')
page.fill('____', '첫 자동화')
print(page.locator('#course-name').get_attribute('value'))
print(page.locator('#memo').get_attribute('value'))
~~~

---

## 문제 5 — 저장 버튼 클릭하기

지시된 값을 코드로 추출하거나 상태를 변경한다.

**기대 결과 형태**: 요구한 값이 한 줄 또는 리스트/딕셔너리 형태로 출력된다.

**빈칸 힌트**: HTML 구조, CSV 헤더, 이전 예제를 확인한 뒤 `____` 부분을 직접 채운다.

~~~python
result = page.click('____')
print(result)
print(page.text_content('____'))
~~~

---

## 문제 6 — data-testid selector 사용하기

지시된 값을 코드로 추출하거나 상태를 변경한다.

**기대 결과 형태**: 요구한 값이 한 줄 또는 리스트/딕셔너리 형태로 출력된다.

**빈칸 힌트**: HTML 구조, CSV 헤더, 이전 예제를 확인한 뒤 `____` 부분을 직접 채운다.

~~~python
page2 = MiniPage(load_text('form_page.html'))
page2.fill('[data-testid="____"]', '이서연')
print(page2.locator('[data-testid="student-name"]').get_attribute('____'))
~~~

---

## 문제 7 — CSV 케이스 읽기

지시된 값을 코드로 추출하거나 상태를 변경한다.

**기대 결과 형태**: 요구한 값이 한 줄 또는 리스트/딕셔너리 형태로 출력된다.

**빈칸 힌트**: HTML 구조, CSV 헤더, 이전 예제를 확인한 뒤 `____` 부분을 직접 채운다.

~~~python
cases = list(csv.DictReader(load_text('____').splitlines()))
print(len(cases))
print(cases[0]['____'])
~~~

---

## 문제 8 — 여러 케이스 폼 자동화

지시된 값을 코드로 추출하거나 상태를 변경한다.

**기대 결과 형태**: 요구한 값이 한 줄 또는 리스트/딕셔너리 형태로 출력된다.

**빈칸 힌트**: HTML 구조, CSV 헤더, 이전 예제를 확인한 뒤 `____` 부분을 직접 채운다.

~~~python
outputs = []
for row in cases:
    p = MiniPage(load_text('form_page.html'))
    p.fill('#student-name', row['____'])
    p.fill('#course-name', row['____'])
    p.fill('#memo', row['____'])
    p.click('#submit-profile')
    outputs.append(p.text_content('____'))
print(outputs[:2])
~~~

---

## 문제 9 — Todo 앱 상태 세기

지시된 값을 코드로 추출하거나 상태를 변경한다.

**기대 결과 형태**: 요구한 값이 한 줄 또는 리스트/딕셔너리 형태로 출력된다.

**빈칸 힌트**: HTML 구조, CSV 헤더, 이전 예제를 확인한 뒤 `____` 부분을 직접 채운다.

~~~python
todo = MiniPage(load_text('____'))
items = todo.locator('li').elements()
counts = {}
for item in items:
    status = item['____']
    counts[status] = counts.get(status, 0) + 1
print(counts)
~~~

---

## 문제 10 — Todo 완료 버튼 클릭

지시된 값을 코드로 추출하거나 상태를 변경한다.

**기대 결과 형태**: 요구한 값이 한 줄 또는 리스트/딕셔너리 형태로 출력된다.

**빈칸 힌트**: HTML 구조, CSV 헤더, 이전 예제를 확인한 뒤 `____` 부분을 직접 채운다.

~~~python
print(todo.locator('____').get_attribute('data-status'))
todo.click('____')
print(todo.locator('____').get_attribute('data-status'))
~~~

---

## 문제 11 — 탭 전환하기

지시된 값을 코드로 추출하거나 상태를 변경한다.

**기대 결과 형태**: 요구한 값이 한 줄 또는 리스트/딕셔너리 형태로 출력된다.

**빈칸 힌트**: HTML 구조, CSV 헤더, 이전 예제를 확인한 뒤 `____` 부분을 직접 채운다.

~~~python
tabs = MiniPage(load_text('____'))
print(tabs.text_content('#panel-python'))
tabs.click('button[data-target="____"]')
print(tabs.text_content('#panel-web'))
~~~

---

## 문제 12 — 행동 로그 확인

지시된 값을 코드로 추출하거나 상태를 변경한다.

**기대 결과 형태**: 요구한 값이 한 줄 또는 리스트/딕셔너리 형태로 출력된다.

**빈칸 힌트**: HTML 구조, CSV 헤더, 이전 예제를 확인한 뒤 `____` 부분을 직접 채운다.

~~~python
log_page = MiniPage(load_text('form_page.html'))
log_page.fill('#student-name', '로그학생')
log_page.click('#submit-profile')
print(log_page.____)
~~~

---

## 문제 13 — 실패 selector 처리

지시된 값을 코드로 추출하거나 상태를 변경한다.

**기대 결과 형태**: 요구한 값이 한 줄 또는 리스트/딕셔너리 형태로 출력된다.

**빈칸 힌트**: HTML 구조, CSV 헤더, 이전 예제를 확인한 뒤 `____` 부분을 직접 채운다.

~~~python
try:
    page.locator('____').text_content()
except Exception as error:
    print(type(error).__name__)
~~~

---

## 문제 14 — QA 결과 리스트 만들기

지시된 값을 코드로 추출하거나 상태를 변경한다.

**기대 결과 형태**: 요구한 값이 한 줄 또는 리스트/딕셔너리 형태로 출력된다.

**빈칸 힌트**: HTML 구조, CSV 헤더, 이전 예제를 확인한 뒤 `____` 부분을 직접 채운다.

~~~python
qa_rows = []
for text in outputs:
    qa_rows.append({'result': text, 'ok': ' / ' in text})
print(qa_rows[0])
print(sum(row['____'] for row in qa_rows))
~~~

---

## 문제 15 — QA 로그 CSV 저장

지시된 값을 코드로 추출하거나 상태를 변경한다.

**기대 결과 형태**: 요구한 값이 한 줄 또는 리스트/딕셔너리 형태로 출력된다.

**빈칸 힌트**: HTML 구조, CSV 헤더, 이전 예제를 확인한 뒤 `____` 부분을 직접 채운다.

~~~python
with open('lesson06_qa_log.csv', 'w', newline='', encoding='utf-8') as f:
    writer = csv.DictWriter(f, fieldnames=['result', 'ok'])
    writer.____()
    writer.____(qa_rows)
print('saved:', len(qa_rows))
~~~

---

## 제출 전 점검

문제를 다 푼 뒤에는 출력만 보지 말고 실행 순서를 다시 확인한다. 폼 자동화는 파일 읽기, selector 확인, 입력, 클릭, 결과 검증 순서로 진행되어야 한다. Todo와 탭 문제는 클릭 전후 같은 대상을 비교했는지 확인한다. 로그와 CSV 저장 문제는 파일 경로가 출력되고 실제 파일이 생성되는지까지 확인한다.

| 항목 | 확인 질문 | 기준 |
|---|---|---|
| selector | id나 data-testid를 사용했는가 | 화면 문구보다 구조 속성 우선 |
| 입력 | fill 이후 value를 확인했는가 | 입력값이 같은 selector에 남아 있음 |
| 클릭 | 클릭 후 result나 data-state를 읽었는가 | 버튼 동작이 상태 변화로 확인됨 |
| 반복 | CSV 모든 행을 처리했는가 | outputs 길이와 cases 길이 일치 |
| 상태 | Todo 클릭 전후 상태가 달라졌는가 | 같은 task id로 비교 |
| 탭 | target panel의 텍스트를 읽었는가 | Web 또는 Report panel 확인 |
| wait | 조건 selector를 기다렸는가 | 무작정 sleep을 쓰지 않음 |
| 저장 | CSV header와 row가 있는가 | 파일 패널에서 확인 가능 |

실제 브라우저 자동화로 확장할 때는 바로 외부 사이트를 반복 요청하지 않는다. 먼저 fixture에서 흐름을 검증하고, 테스트 계정과 요청 간격, 개인정보 저장 여부를 확인한다. 수업 제출물에는 실제 비밀번호나 개인 정보가 들어가지 않아야 한다.
