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

# 레슨 06 — 실습 문제 정답지

[![Open in Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/Kevin-innovation/jupyter-lecture/blob/main/python-web-automation/lectures/06/%EB%A0%88%EC%8A%A8%2006%20%E2%80%94%20%EB%B8%8C%EB%9D%BC%EC%9A%B0%EC%A0%80%20%EC%9E%90%EB%8F%99%ED%99%94%20%EC%9E%85%EB%AC%B8.ipynb)

> 🔒 교사·관리자 전용. 학생에게 배포 금지.

브라우저 자동화 입문 실습 문제의 모범 답안이다. 출력값만 보지 말고 상태 변화, selector 안정성, 로그를 함께 확인한다.

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

## 문제 1 정답 — 폼 페이지 제목 읽기

~~~python
page = MiniPage(load_text('form_page.html'))
print(page.text_content('h1'))
~~~

### 왜 이 코드가 정답인지

이 답안은 문제에서 요구한 상태나 화면 구조를 먼저 명확한 자료구조로 바꾼 뒤 필요한 값만 선택한다. MiniPage가 실제 브라우저 page처럼 locator와 text_content 흐름을 제공한다. 결과가 맞아도 세션 상태, locator, wait, 로그가 코드에 남아 있지 않으면 운영형 자동화로 보기 어렵다.

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

## 문제 2 정답 — input 개수 세기

~~~python
print(page.locator('input').count())
~~~

### 왜 이 코드가 정답인지

이 답안은 문제에서 요구한 상태나 화면 구조를 먼저 명확한 자료구조로 바꾼 뒤 필요한 값만 선택한다. 폼 자동화 전에 입력 칸 개수를 확인한다. 결과가 맞아도 세션 상태, locator, wait, 로그가 코드에 남아 있지 않으면 운영형 자동화로 보기 어렵다.

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

## 문제 3 정답 — 이름 입력하기

~~~python
page.fill('#student-name', '김도윤')
print(page.locator('#student-name').get_attribute('value'))
~~~

### 왜 이 코드가 정답인지

이 답안은 문제에서 요구한 상태나 화면 구조를 먼저 명확한 자료구조로 바꾼 뒤 필요한 값만 선택한다. fill은 화면 상태를 바꾸는 동작이므로 value 속성을 확인한다. 결과가 맞아도 세션 상태, locator, wait, 로그가 코드에 남아 있지 않으면 운영형 자동화로 보기 어렵다.

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

## 문제 4 정답 — 과정과 메모 입력하기

~~~python
page.fill('#course-name', 'Python')
page.fill('#memo', '첫 자동화')
print(page.locator('#course-name').get_attribute('value'))
print(page.locator('#memo').get_attribute('value'))
~~~

### 왜 이 코드가 정답인지

이 답안은 문제에서 요구한 상태나 화면 구조를 먼저 명확한 자료구조로 바꾼 뒤 필요한 값만 선택한다. 여러 입력은 selector가 서로 섞이지 않아야 한다. 결과가 맞아도 세션 상태, locator, wait, 로그가 코드에 남아 있지 않으면 운영형 자동화로 보기 어렵다.

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

## 문제 5 정답 — 저장 버튼 클릭하기

~~~python
result = page.click('#submit-profile')
print(result)
print(page.text_content('#result'))
~~~

### 왜 이 코드가 정답인지

이 답안은 문제에서 요구한 상태나 화면 구조를 먼저 명확한 자료구조로 바꾼 뒤 필요한 값만 선택한다. click 후 output이 바뀌어야 자동화가 성공한 것이다. 결과가 맞아도 세션 상태, locator, wait, 로그가 코드에 남아 있지 않으면 운영형 자동화로 보기 어렵다.

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

## 문제 6 정답 — data-testid selector 사용하기

~~~python
page2 = MiniPage(load_text('form_page.html'))
page2.fill('[data-testid="student-name"]', '이서연')
print(page2.locator('[data-testid="student-name"]').get_attribute('value'))
~~~

### 왜 이 코드가 정답인지

이 답안은 문제에서 요구한 상태나 화면 구조를 먼저 명확한 자료구조로 바꾼 뒤 필요한 값만 선택한다. data-testid는 문구나 CSS 변경보다 안정적인 테스트 selector다. 결과가 맞아도 세션 상태, locator, wait, 로그가 코드에 남아 있지 않으면 운영형 자동화로 보기 어렵다.

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

## 문제 7 정답 — CSV 케이스 읽기

~~~python
cases = list(csv.DictReader(load_text('form_cases.csv').splitlines()))
print(len(cases))
print(cases[0]['name'])
~~~

### 왜 이 코드가 정답인지

이 답안은 문제에서 요구한 상태나 화면 구조를 먼저 명확한 자료구조로 바꾼 뒤 필요한 값만 선택한다. 여러 입력 자동화는 CSV 케이스와 결합하면 반복 테스트가 된다. 결과가 맞아도 세션 상태, locator, wait, 로그가 코드에 남아 있지 않으면 운영형 자동화로 보기 어렵다.

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

## 문제 8 정답 — 여러 케이스 폼 자동화

~~~python
outputs = []
for row in cases:
    p = MiniPage(load_text('form_page.html'))
    p.fill('#student-name', row['name'])
    p.fill('#course-name', row['course'])
    p.fill('#memo', row['memo'])
    p.click('#submit-profile')
    outputs.append(p.text_content('#result'))
print(outputs[:2])
~~~

### 왜 이 코드가 정답인지

이 답안은 문제에서 요구한 상태나 화면 구조를 먼저 명확한 자료구조로 바꾼 뒤 필요한 값만 선택한다. CSV 행마다 새 page를 만들면 케이스 간 상태가 섞이지 않는다. 결과가 맞아도 세션 상태, locator, wait, 로그가 코드에 남아 있지 않으면 운영형 자동화로 보기 어렵다.

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

## 문제 9 정답 — Todo 앱 상태 세기

~~~python
todo = MiniPage(load_text('todo_app.html'))
items = todo.locator('li').elements()
counts = {}
for item in items:
    status = item['data-status']
    counts[status] = counts.get(status, 0) + 1
print(counts)
~~~

### 왜 이 코드가 정답인지

이 답안은 문제에서 요구한 상태나 화면 구조를 먼저 명확한 자료구조로 바꾼 뒤 필요한 값만 선택한다. 클릭 전 상태를 세어야 클릭 후 변화 검증이 가능하다. 결과가 맞아도 세션 상태, locator, wait, 로그가 코드에 남아 있지 않으면 운영형 자동화로 보기 어렵다.

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

## 문제 10 정답 — Todo 완료 버튼 클릭

~~~python
print(todo.locator('#task-1').get_attribute('data-status'))
todo.click('#task-1 button')
print(todo.locator('#task-1').get_attribute('data-status'))
~~~

### 왜 이 코드가 정답인지

이 답안은 문제에서 요구한 상태나 화면 구조를 먼저 명확한 자료구조로 바꾼 뒤 필요한 값만 선택한다. 버튼 클릭이 대상 li의 data-status를 바꾸는지 확인한다. 결과가 맞아도 세션 상태, locator, wait, 로그가 코드에 남아 있지 않으면 운영형 자동화로 보기 어렵다.

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

## 문제 11 정답 — 탭 전환하기

~~~python
tabs = MiniPage(load_text('tabs_page.html'))
print(tabs.text_content('#panel-python'))
tabs.click('button[data-target="panel-web"]')
print(tabs.text_content('#panel-web'))
~~~

### 왜 이 코드가 정답인지

이 답안은 문제에서 요구한 상태나 화면 구조를 먼저 명확한 자료구조로 바꾼 뒤 필요한 값만 선택한다. 탭 UI는 버튼 클릭 후 표시되는 panel이 바뀌는 흐름이다. 결과가 맞아도 세션 상태, locator, wait, 로그가 코드에 남아 있지 않으면 운영형 자동화로 보기 어렵다.

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

## 문제 12 정답 — 행동 로그 확인

~~~python
log_page = MiniPage(load_text('form_page.html'))
log_page.fill('#student-name', '로그학생')
log_page.click('#submit-profile')
print(log_page.log)
~~~

### 왜 이 코드가 정답인지

이 답안은 문제에서 요구한 상태나 화면 구조를 먼저 명확한 자료구조로 바꾼 뒤 필요한 값만 선택한다. 자동화는 수행한 action을 로그로 남기면 재현성이 좋아진다. 결과가 맞아도 세션 상태, locator, wait, 로그가 코드에 남아 있지 않으면 운영형 자동화로 보기 어렵다.

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

## 문제 13 정답 — 실패 selector 처리

~~~python
try:
    page.locator('.missing').text_content()
except Exception as error:
    print(type(error).__name__)
~~~

### 왜 이 코드가 정답인지

이 답안은 문제에서 요구한 상태나 화면 구조를 먼저 명확한 자료구조로 바꾼 뒤 필요한 값만 선택한다. selector가 틀릴 수 있으므로 실패를 관찰하고 수정해야 한다. 결과가 맞아도 세션 상태, locator, wait, 로그가 코드에 남아 있지 않으면 운영형 자동화로 보기 어렵다.

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

## 문제 14 정답 — QA 결과 리스트 만들기

~~~python
qa_rows = []
for text in outputs:
    qa_rows.append({'result': text, 'ok': ' / ' in text})
print(qa_rows[0])
print(sum(row['ok'] for row in qa_rows))
~~~

### 왜 이 코드가 정답인지

이 답안은 문제에서 요구한 상태나 화면 구조를 먼저 명확한 자료구조로 바꾼 뒤 필요한 값만 선택한다. 자동화 결과는 사람이 볼 문자열과 통과 여부를 함께 남긴다. 결과가 맞아도 세션 상태, locator, wait, 로그가 코드에 남아 있지 않으면 운영형 자동화로 보기 어렵다.

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

## 문제 15 정답 — QA 로그 CSV 저장

~~~python
with open('lesson06_qa_log.csv', 'w', newline='', encoding='utf-8') as f:
    writer = csv.DictWriter(f, fieldnames=['result', 'ok'])
    writer.writeheader()
    writer.writerows(qa_rows)
print('saved:', len(qa_rows))
~~~

### 왜 이 코드가 정답인지

이 답안은 문제에서 요구한 상태나 화면 구조를 먼저 명확한 자료구조로 바꾼 뒤 필요한 값만 선택한다. 여러 케이스 실행 결과를 CSV로 저장해야 검수와 공유가 가능하다. 결과가 맞아도 세션 상태, locator, wait, 로그가 코드에 남아 있지 않으면 운영형 자동화로 보기 어렵다.

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
