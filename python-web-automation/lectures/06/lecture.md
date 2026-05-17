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

# 레슨 06 — 브라우저 자동화 입문

[![Open in Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/Kevin-innovation/jupyter-lecture/blob/main/python-web-automation/lectures/06/[학생용] 레슨 06 — 브라우저 자동화 입문.ipynb)

이 노트북은 읽기와 따라하기용 강의 노트북이다. 학생은 셀을 위에서 아래로 실행하며 웹 자동화에서 상태가 어떻게 유지되고 화면 조작이 어떤 순서로 기록되는지 확인한다. 브라우저 자동화 입문는 실제 사이트 대신 합성 fixture로 안전하게 연습한다.

## 학습 목표

1. 브라우저 자동화가 필요한 상황을 구분한다.
2. locator, fill, click, text_content 흐름을 이해한다.
3. 폼 입력과 버튼 클릭 결과를 검증한다.
4. 여러 CSV 케이스로 반복 QA를 수행한다.
5. 자동화 로그를 CSV로 저장한다.

---

## 1. 브라우저 자동화 흐름

브라우저 자동화는 page를 열고 locator로 요소를 찾고 fill/click으로 상태를 바꾼 뒤 결과를 확인하는 흐름이다. 실제 Playwright 문법과 같은 개념을 수업용 MiniPage로 재현한다.

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

page = MiniPage(load_text('form_page.html'))
print(page.text_content('h1'))
~~~

---

## 2. 폼 입력과 클릭

입력은 value 상태를 바꾸고 클릭은 output 상태를 바꾼다.

~~~python
page.fill('#student-name', '김도윤')
page.fill('#course-name', 'Python')
page.fill('#memo', '첫 자동화')
page.click('#submit-profile')
print(page.text_content('#result'))
~~~

---

## 3. 반복 QA

CSV 케이스를 이용하면 같은 폼을 여러 입력으로 반복 검증할 수 있다.

~~~python
cases = list(csv.DictReader(load_text('form_cases.csv').splitlines()))
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

---

## 데이터 출처와 안전 규칙

이 레슨의 파일은 모두 수업용 합성 데이터다. 실제 사이트의 개인정보, 로그인 정보, 유료 콘텐츠를 포함하지 않는다. 실제 사이트로 확장할 때는 robots.txt, 이용 약관, 요청 간격, 개인정보 여부를 먼저 확인한다. 수업 중에는 fixture를 반복 실행하며 구조를 익히고, 외부 사이트를 빠르게 반복 요청하지 않는다.

### 보강 설명 1

레슨 06 강의은 실행 결과만 맞추는 것이 아니라 재현 가능한 절차를 남기는 것이 핵심이다. 입력 파일, 반복 단위, selector 또는 상태 변수, 저장 경로를 분리하면 오류 위치를 빠르게 찾을 수 있다.

### 보강 설명 2

학생이 막히면 완성 코드를 보여주기보다 HTML 구조, CSV 헤더, 상태 변화 순서를 먼저 말로 설명하게 한다. 구조를 설명할 수 있으면 코드도 안정된다.

### 보강 설명 3

실제 자동화는 다음 주에도 다시 실행되어야 한다. 그래서 쿠키, 폼 입력, selector, wait, 로그 같은 운영 요소를 초반부터 명시적으로 다룬다.

### 보강 설명 4

외부 사이트로 확장할 때는 약관, robots.txt, 요청 간격, 개인정보 여부를 먼저 확인한다. 이 코스의 fixture는 안전한 반복 연습을 위한 합성 데이터다.

### 보강 설명 5

레슨 06 강의은 실행 결과만 맞추는 것이 아니라 재현 가능한 절차를 남기는 것이 핵심이다. 입력 파일, 반복 단위, selector 또는 상태 변수, 저장 경로를 분리하면 오류 위치를 빠르게 찾을 수 있다.

### 보강 설명 6

학생이 막히면 완성 코드를 보여주기보다 HTML 구조, CSV 헤더, 상태 변화 순서를 먼저 말로 설명하게 한다. 구조를 설명할 수 있으면 코드도 안정된다.

### 보강 설명 7

실제 자동화는 다음 주에도 다시 실행되어야 한다. 그래서 쿠키, 폼 입력, selector, wait, 로그 같은 운영 요소를 초반부터 명시적으로 다룬다.

### 보강 설명 8

외부 사이트로 확장할 때는 약관, robots.txt, 요청 간격, 개인정보 여부를 먼저 확인한다. 이 코스의 fixture는 안전한 반복 연습을 위한 합성 데이터다.

### 보강 설명 9

레슨 06 강의은 실행 결과만 맞추는 것이 아니라 재현 가능한 절차를 남기는 것이 핵심이다. 입력 파일, 반복 단위, selector 또는 상태 변수, 저장 경로를 분리하면 오류 위치를 빠르게 찾을 수 있다.

### 보강 설명 10

학생이 막히면 완성 코드를 보여주기보다 HTML 구조, CSV 헤더, 상태 변화 순서를 먼저 말로 설명하게 한다. 구조를 설명할 수 있으면 코드도 안정된다.

### 보강 설명 11

실제 자동화는 다음 주에도 다시 실행되어야 한다. 그래서 쿠키, 폼 입력, selector, wait, 로그 같은 운영 요소를 초반부터 명시적으로 다룬다.

### 보강 설명 12

외부 사이트로 확장할 때는 약관, robots.txt, 요청 간격, 개인정보 여부를 먼저 확인한다. 이 코스의 fixture는 안전한 반복 연습을 위한 합성 데이터다.

### 보강 설명 13

레슨 06 강의은 실행 결과만 맞추는 것이 아니라 재현 가능한 절차를 남기는 것이 핵심이다. 입력 파일, 반복 단위, selector 또는 상태 변수, 저장 경로를 분리하면 오류 위치를 빠르게 찾을 수 있다.

### 보강 설명 14

학생이 막히면 완성 코드를 보여주기보다 HTML 구조, CSV 헤더, 상태 변화 순서를 먼저 말로 설명하게 한다. 구조를 설명할 수 있으면 코드도 안정된다.

### 보강 설명 15

실제 자동화는 다음 주에도 다시 실행되어야 한다. 그래서 쿠키, 폼 입력, selector, wait, 로그 같은 운영 요소를 초반부터 명시적으로 다룬다.

### 보강 설명 16

외부 사이트로 확장할 때는 약관, robots.txt, 요청 간격, 개인정보 여부를 먼저 확인한다. 이 코스의 fixture는 안전한 반복 연습을 위한 합성 데이터다.

### 보강 설명 17

레슨 06 강의은 실행 결과만 맞추는 것이 아니라 재현 가능한 절차를 남기는 것이 핵심이다. 입력 파일, 반복 단위, selector 또는 상태 변수, 저장 경로를 분리하면 오류 위치를 빠르게 찾을 수 있다.

### 보강 설명 18

학생이 막히면 완성 코드를 보여주기보다 HTML 구조, CSV 헤더, 상태 변화 순서를 먼저 말로 설명하게 한다. 구조를 설명할 수 있으면 코드도 안정된다.

### 보강 설명 19

실제 자동화는 다음 주에도 다시 실행되어야 한다. 그래서 쿠키, 폼 입력, selector, wait, 로그 같은 운영 요소를 초반부터 명시적으로 다룬다.

### 보강 설명 20

외부 사이트로 확장할 때는 약관, robots.txt, 요청 간격, 개인정보 여부를 먼저 확인한다. 이 코스의 fixture는 안전한 반복 연습을 위한 합성 데이터다.

### 보강 설명 21

레슨 06 강의은 실행 결과만 맞추는 것이 아니라 재현 가능한 절차를 남기는 것이 핵심이다. 입력 파일, 반복 단위, selector 또는 상태 변수, 저장 경로를 분리하면 오류 위치를 빠르게 찾을 수 있다.

### 보강 설명 22

학생이 막히면 완성 코드를 보여주기보다 HTML 구조, CSV 헤더, 상태 변화 순서를 먼저 말로 설명하게 한다. 구조를 설명할 수 있으면 코드도 안정된다.

### 보강 설명 23

실제 자동화는 다음 주에도 다시 실행되어야 한다. 그래서 쿠키, 폼 입력, selector, wait, 로그 같은 운영 요소를 초반부터 명시적으로 다룬다.

### 보강 설명 24

외부 사이트로 확장할 때는 약관, robots.txt, 요청 간격, 개인정보 여부를 먼저 확인한다. 이 코스의 fixture는 안전한 반복 연습을 위한 합성 데이터다.

### 보강 설명 25

레슨 06 강의은 실행 결과만 맞추는 것이 아니라 재현 가능한 절차를 남기는 것이 핵심이다. 입력 파일, 반복 단위, selector 또는 상태 변수, 저장 경로를 분리하면 오류 위치를 빠르게 찾을 수 있다.

### 보강 설명 26

학생이 막히면 완성 코드를 보여주기보다 HTML 구조, CSV 헤더, 상태 변화 순서를 먼저 말로 설명하게 한다. 구조를 설명할 수 있으면 코드도 안정된다.

### 보강 설명 27

실제 자동화는 다음 주에도 다시 실행되어야 한다. 그래서 쿠키, 폼 입력, selector, wait, 로그 같은 운영 요소를 초반부터 명시적으로 다룬다.

### 보강 설명 28

외부 사이트로 확장할 때는 약관, robots.txt, 요청 간격, 개인정보 여부를 먼저 확인한다. 이 코스의 fixture는 안전한 반복 연습을 위한 합성 데이터다.

### 보강 설명 29

레슨 06 강의은 실행 결과만 맞추는 것이 아니라 재현 가능한 절차를 남기는 것이 핵심이다. 입력 파일, 반복 단위, selector 또는 상태 변수, 저장 경로를 분리하면 오류 위치를 빠르게 찾을 수 있다.

### 보강 설명 30

학생이 막히면 완성 코드를 보여주기보다 HTML 구조, CSV 헤더, 상태 변화 순서를 먼저 말로 설명하게 한다. 구조를 설명할 수 있으면 코드도 안정된다.

### 보강 설명 31

실제 자동화는 다음 주에도 다시 실행되어야 한다. 그래서 쿠키, 폼 입력, selector, wait, 로그 같은 운영 요소를 초반부터 명시적으로 다룬다.

### 보강 설명 32

외부 사이트로 확장할 때는 약관, robots.txt, 요청 간격, 개인정보 여부를 먼저 확인한다. 이 코스의 fixture는 안전한 반복 연습을 위한 합성 데이터다.

### 보강 설명 33

레슨 06 강의은 실행 결과만 맞추는 것이 아니라 재현 가능한 절차를 남기는 것이 핵심이다. 입력 파일, 반복 단위, selector 또는 상태 변수, 저장 경로를 분리하면 오류 위치를 빠르게 찾을 수 있다.

### 보강 설명 34

학생이 막히면 완성 코드를 보여주기보다 HTML 구조, CSV 헤더, 상태 변화 순서를 먼저 말로 설명하게 한다. 구조를 설명할 수 있으면 코드도 안정된다.

### 보강 설명 35

실제 자동화는 다음 주에도 다시 실행되어야 한다. 그래서 쿠키, 폼 입력, selector, wait, 로그 같은 운영 요소를 초반부터 명시적으로 다룬다.

### 보강 설명 36

외부 사이트로 확장할 때는 약관, robots.txt, 요청 간격, 개인정보 여부를 먼저 확인한다. 이 코스의 fixture는 안전한 반복 연습을 위한 합성 데이터다.

### 보강 설명 37

레슨 06 강의은 실행 결과만 맞추는 것이 아니라 재현 가능한 절차를 남기는 것이 핵심이다. 입력 파일, 반복 단위, selector 또는 상태 변수, 저장 경로를 분리하면 오류 위치를 빠르게 찾을 수 있다.

### 보강 설명 38

학생이 막히면 완성 코드를 보여주기보다 HTML 구조, CSV 헤더, 상태 변화 순서를 먼저 말로 설명하게 한다. 구조를 설명할 수 있으면 코드도 안정된다.

### 보강 설명 39

실제 자동화는 다음 주에도 다시 실행되어야 한다. 그래서 쿠키, 폼 입력, selector, wait, 로그 같은 운영 요소를 초반부터 명시적으로 다룬다.

### 보강 설명 40

외부 사이트로 확장할 때는 약관, robots.txt, 요청 간격, 개인정보 여부를 먼저 확인한다. 이 코스의 fixture는 안전한 반복 연습을 위한 합성 데이터다.

### 보강 설명 41

레슨 06 강의은 실행 결과만 맞추는 것이 아니라 재현 가능한 절차를 남기는 것이 핵심이다. 입력 파일, 반복 단위, selector 또는 상태 변수, 저장 경로를 분리하면 오류 위치를 빠르게 찾을 수 있다.

### 보강 설명 42

학생이 막히면 완성 코드를 보여주기보다 HTML 구조, CSV 헤더, 상태 변화 순서를 먼저 말로 설명하게 한다. 구조를 설명할 수 있으면 코드도 안정된다.

### 보강 설명 43

실제 자동화는 다음 주에도 다시 실행되어야 한다. 그래서 쿠키, 폼 입력, selector, wait, 로그 같은 운영 요소를 초반부터 명시적으로 다룬다.

### 보강 설명 44

외부 사이트로 확장할 때는 약관, robots.txt, 요청 간격, 개인정보 여부를 먼저 확인한다. 이 코스의 fixture는 안전한 반복 연습을 위한 합성 데이터다.

### 보강 설명 45

레슨 06 강의은 실행 결과만 맞추는 것이 아니라 재현 가능한 절차를 남기는 것이 핵심이다. 입력 파일, 반복 단위, selector 또는 상태 변수, 저장 경로를 분리하면 오류 위치를 빠르게 찾을 수 있다.

### 보강 설명 46

학생이 막히면 완성 코드를 보여주기보다 HTML 구조, CSV 헤더, 상태 변화 순서를 먼저 말로 설명하게 한다. 구조를 설명할 수 있으면 코드도 안정된다.
