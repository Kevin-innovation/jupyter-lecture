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

---

## 강의 보강 노트

이 절은 수업 중 교사가 질문으로 풀어낼 수 있는 운영형 설명이다. 학생이 셀을 실행한 뒤 결과만 맞히지 않고 자동화 절차를 말로 설명하도록 돕는다.

### 1. 브라우저 자동화가 필요한 순간

정적 HTML 파싱으로 해결되는 문제에 브라우저 자동화를 쓰면 느리고 불안정해진다. 반대로 클릭, 입력, 탭 전환, 동적 결과 확인이 필요하면 브라우저 모델이 더 자연스럽다.

### 2. MiniPage의 목적

이 레슨의 MiniPage는 실제 Playwright를 그대로 흉내 내는 도구가 아니라 클릭과 입력의 순서를 안전하게 연습하는 모델이다. 외부 브라우저 설치 실패 없이 수업 흐름을 유지하기 위한 장치다.

### 3. 입력값과 결과 영역

폼 자동화에서 학생이 자주 놓치는 부분은 입력 후 결과 영역이 정말 바뀌었는지 확인하지 않는 것이다. fill, click, result 확인을 한 묶음으로 작성하게 한다.

### 4. 탭과 상태

탭 전환은 보이는 화면만 바꾸는 것이 아니라 hidden 속성이나 aria 상태를 조정한다. 학생에게 현재 보이는 panel이 무엇인지 selector로 확인하게 하면 UI 자동화의 구조를 이해할 수 있다.

### 5. 로그의 역할

브라우저 자동화는 화면이 바뀌기 때문에 나중에 어떤 버튼을 눌렀는지 기억하기 어렵다. action, selector, value, result를 로그로 남기면 실패 재현이 쉬워진다.

### 6. 실제 도구로 확장

Playwright나 Selenium으로 넘어가도 개념은 같다. locator를 잡고, 값을 채우고, 클릭하고, 결과를 기다리고, 실패를 기록하는 순서만 유지하면 도구 문법은 바뀌어도 흐름은 유지된다.

### 7. 채점 관점

학생 답안은 버튼을 누른 결과만 보지 말고 클릭 전후 상태와 로그 길이를 함께 본다. 자동화는 보이지 않는 절차가 중요하므로 결과 문자열 하나만 맞아도 절차가 비어 있으면 보완한다.

### 체크포인트 1: 검증 기준 설명한다

다음 셀에서 재사용하기 위해, 레슨 06 브라우저 자동화 입문에서는 검증 기준을/를 설명한다는 과정을 별도의 단계로 둔다. 이때 학생은 화면에 보이는 값 하나보다 어떤 입력에서 어떤 변환을 거쳐 결과가 나왔는지 설명해야 한다. 강사는 변수명, 출력 형태, 저장 여부를 함께 확인하며 다음 실행에서도 같은 결과가 나오는지 묻는다.

### 체크포인트 2: 중간 변수 저장한다

실제 사이트 확장 전에 위험을 낮추기 위해, 레슨 06 브라우저 자동화 입문에서는 중간 변수을/를 저장한다는 과정을 별도의 단계로 둔다. 이때 학생은 화면에 보이는 값 하나보다 어떤 입력에서 어떤 변환을 거쳐 결과가 나왔는지 설명해야 한다. 강사는 변수명, 출력 형태, 저장 여부를 함께 확인하며 다음 실행에서도 같은 결과가 나오는지 묻는다.

### 체크포인트 3: 입력 파일 되돌아본다

저장 파일의 신뢰도를 높이기 위해, 레슨 06 브라우저 자동화 입문에서는 입력 파일을/를 되돌아본다는 과정을 별도의 단계로 둔다. 이때 학생은 화면에 보이는 값 하나보다 어떤 입력에서 어떤 변환을 거쳐 결과가 나왔는지 설명해야 한다. 강사는 변수명, 출력 형태, 저장 여부를 함께 확인하며 다음 실행에서도 같은 결과가 나오는지 묻는다.

### 체크포인트 4: 선택자 표준화한다

코랩과 로컬 실행 차이를 줄이기 위해, 레슨 06 브라우저 자동화 입문에서는 선택자을/를 표준화한다는 과정을 별도의 단계로 둔다. 이때 학생은 화면에 보이는 값 하나보다 어떤 입력에서 어떤 변환을 거쳐 결과가 나왔는지 설명해야 한다. 강사는 변수명, 출력 형태, 저장 여부를 함께 확인하며 다음 실행에서도 같은 결과가 나오는지 묻는다.

### 체크포인트 5: 오류 메시지 검증한다

수업 중 피드백 시간을 줄이기 위해, 레슨 06 브라우저 자동화 입문에서는 오류 메시지을/를 검증한다는 과정을 별도의 단계로 둔다. 이때 학생은 화면에 보이는 값 하나보다 어떤 입력에서 어떤 변환을 거쳐 결과가 나왔는지 설명해야 한다. 강사는 변수명, 출력 형태, 저장 여부를 함께 확인하며 다음 실행에서도 같은 결과가 나오는지 묻는다.

### 체크포인트 6: 결과 행 수 확인한다

학생이 막힌 지점을 찾기 위해, 레슨 06 브라우저 자동화 입문에서는 결과 행 수을/를 확인한다는 과정을 별도의 단계로 둔다. 이때 학생은 화면에 보이는 값 하나보다 어떤 입력에서 어떤 변환을 거쳐 결과가 나왔는지 설명해야 한다. 강사는 변수명, 출력 형태, 저장 여부를 함께 확인하며 다음 실행에서도 같은 결과가 나오는지 묻는다.

### 체크포인트 7: 검증 기준 분리한다

운영자가 결과를 이해할 수 있게, 레슨 06 브라우저 자동화 입문에서는 검증 기준을/를 분리한다는 과정을 별도의 단계로 둔다. 이때 학생은 화면에 보이는 값 하나보다 어떤 입력에서 어떤 변환을 거쳐 결과가 나왔는지 설명해야 한다. 강사는 변수명, 출력 형태, 저장 여부를 함께 확인하며 다음 실행에서도 같은 결과가 나오는지 묻는다.

### 체크포인트 8: 중간 변수 기록한다

재실행했을 때 같은 결과를 얻기 위해, 레슨 06 브라우저 자동화 입문에서는 중간 변수을/를 기록한다는 과정을 별도의 단계로 둔다. 이때 학생은 화면에 보이는 값 하나보다 어떤 입력에서 어떤 변환을 거쳐 결과가 나왔는지 설명해야 한다. 강사는 변수명, 출력 형태, 저장 여부를 함께 확인하며 다음 실행에서도 같은 결과가 나오는지 묻는다.

### 체크포인트 9: 입력 파일 비교한다

다음 셀에서 재사용하기 위해, 레슨 06 브라우저 자동화 입문에서는 입력 파일을/를 비교한다는 과정을 별도의 단계로 둔다. 이때 학생은 화면에 보이는 값 하나보다 어떤 입력에서 어떤 변환을 거쳐 결과가 나왔는지 설명해야 한다. 강사는 변수명, 출력 형태, 저장 여부를 함께 확인하며 다음 실행에서도 같은 결과가 나오는지 묻는다.

### 체크포인트 10: 선택자 요약한다

실제 사이트 확장 전에 위험을 낮추기 위해, 레슨 06 브라우저 자동화 입문에서는 선택자을/를 요약한다는 과정을 별도의 단계로 둔다. 이때 학생은 화면에 보이는 값 하나보다 어떤 입력에서 어떤 변환을 거쳐 결과가 나왔는지 설명해야 한다. 강사는 변수명, 출력 형태, 저장 여부를 함께 확인하며 다음 실행에서도 같은 결과가 나오는지 묻는다.

### 체크포인트 11: 오류 메시지 검토한다

저장 파일의 신뢰도를 높이기 위해, 레슨 06 브라우저 자동화 입문에서는 오류 메시지을/를 검토한다는 과정을 별도의 단계로 둔다. 이때 학생은 화면에 보이는 값 하나보다 어떤 입력에서 어떤 변환을 거쳐 결과가 나왔는지 설명해야 한다. 강사는 변수명, 출력 형태, 저장 여부를 함께 확인하며 다음 실행에서도 같은 결과가 나오는지 묻는다.

### 체크포인트 12: 결과 행 수 정리한다

코랩과 로컬 실행 차이를 줄이기 위해, 레슨 06 브라우저 자동화 입문에서는 결과 행 수을/를 정리한다는 과정을 별도의 단계로 둔다. 이때 학생은 화면에 보이는 값 하나보다 어떤 입력에서 어떤 변환을 거쳐 결과가 나왔는지 설명해야 한다. 강사는 변수명, 출력 형태, 저장 여부를 함께 확인하며 다음 실행에서도 같은 결과가 나오는지 묻는다.

### 체크포인트 13: 검증 기준 설명한다

수업 중 피드백 시간을 줄이기 위해, 레슨 06 브라우저 자동화 입문에서는 검증 기준을/를 설명한다는 과정을 별도의 단계로 둔다. 이때 학생은 화면에 보이는 값 하나보다 어떤 입력에서 어떤 변환을 거쳐 결과가 나왔는지 설명해야 한다. 강사는 변수명, 출력 형태, 저장 여부를 함께 확인하며 다음 실행에서도 같은 결과가 나오는지 묻는다.

### 체크포인트 14: 중간 변수 저장한다

학생이 막힌 지점을 찾기 위해, 레슨 06 브라우저 자동화 입문에서는 중간 변수을/를 저장한다는 과정을 별도의 단계로 둔다. 이때 학생은 화면에 보이는 값 하나보다 어떤 입력에서 어떤 변환을 거쳐 결과가 나왔는지 설명해야 한다. 강사는 변수명, 출력 형태, 저장 여부를 함께 확인하며 다음 실행에서도 같은 결과가 나오는지 묻는다.

### 체크포인트 15: 입력 파일 되돌아본다

운영자가 결과를 이해할 수 있게, 레슨 06 브라우저 자동화 입문에서는 입력 파일을/를 되돌아본다는 과정을 별도의 단계로 둔다. 이때 학생은 화면에 보이는 값 하나보다 어떤 입력에서 어떤 변환을 거쳐 결과가 나왔는지 설명해야 한다. 강사는 변수명, 출력 형태, 저장 여부를 함께 확인하며 다음 실행에서도 같은 결과가 나오는지 묻는다.

### 체크포인트 16: 선택자 표준화한다

재실행했을 때 같은 결과를 얻기 위해, 레슨 06 브라우저 자동화 입문에서는 선택자을/를 표준화한다는 과정을 별도의 단계로 둔다. 이때 학생은 화면에 보이는 값 하나보다 어떤 입력에서 어떤 변환을 거쳐 결과가 나왔는지 설명해야 한다. 강사는 변수명, 출력 형태, 저장 여부를 함께 확인하며 다음 실행에서도 같은 결과가 나오는지 묻는다.

### 체크포인트 17: 오류 메시지 검증한다

다음 셀에서 재사용하기 위해, 레슨 06 브라우저 자동화 입문에서는 오류 메시지을/를 검증한다는 과정을 별도의 단계로 둔다. 이때 학생은 화면에 보이는 값 하나보다 어떤 입력에서 어떤 변환을 거쳐 결과가 나왔는지 설명해야 한다. 강사는 변수명, 출력 형태, 저장 여부를 함께 확인하며 다음 실행에서도 같은 결과가 나오는지 묻는다.

### 체크포인트 18: 결과 행 수 확인한다

실제 사이트 확장 전에 위험을 낮추기 위해, 레슨 06 브라우저 자동화 입문에서는 결과 행 수을/를 확인한다는 과정을 별도의 단계로 둔다. 이때 학생은 화면에 보이는 값 하나보다 어떤 입력에서 어떤 변환을 거쳐 결과가 나왔는지 설명해야 한다. 강사는 변수명, 출력 형태, 저장 여부를 함께 확인하며 다음 실행에서도 같은 결과가 나오는지 묻는다.

### 체크포인트 19: 검증 기준 분리한다

저장 파일의 신뢰도를 높이기 위해, 레슨 06 브라우저 자동화 입문에서는 검증 기준을/를 분리한다는 과정을 별도의 단계로 둔다. 이때 학생은 화면에 보이는 값 하나보다 어떤 입력에서 어떤 변환을 거쳐 결과가 나왔는지 설명해야 한다. 강사는 변수명, 출력 형태, 저장 여부를 함께 확인하며 다음 실행에서도 같은 결과가 나오는지 묻는다.

### 체크포인트 20: 중간 변수 기록한다

코랩과 로컬 실행 차이를 줄이기 위해, 레슨 06 브라우저 자동화 입문에서는 중간 변수을/를 기록한다는 과정을 별도의 단계로 둔다. 이때 학생은 화면에 보이는 값 하나보다 어떤 입력에서 어떤 변환을 거쳐 결과가 나왔는지 설명해야 한다. 강사는 변수명, 출력 형태, 저장 여부를 함께 확인하며 다음 실행에서도 같은 결과가 나오는지 묻는다.

### 체크포인트 21: 입력 파일 비교한다

수업 중 피드백 시간을 줄이기 위해, 레슨 06 브라우저 자동화 입문에서는 입력 파일을/를 비교한다는 과정을 별도의 단계로 둔다. 이때 학생은 화면에 보이는 값 하나보다 어떤 입력에서 어떤 변환을 거쳐 결과가 나왔는지 설명해야 한다. 강사는 변수명, 출력 형태, 저장 여부를 함께 확인하며 다음 실행에서도 같은 결과가 나오는지 묻는다.

### 체크포인트 22: 선택자 요약한다

학생이 막힌 지점을 찾기 위해, 레슨 06 브라우저 자동화 입문에서는 선택자을/를 요약한다는 과정을 별도의 단계로 둔다. 이때 학생은 화면에 보이는 값 하나보다 어떤 입력에서 어떤 변환을 거쳐 결과가 나왔는지 설명해야 한다. 강사는 변수명, 출력 형태, 저장 여부를 함께 확인하며 다음 실행에서도 같은 결과가 나오는지 묻는다.

### 체크포인트 23: 오류 메시지 검토한다

운영자가 결과를 이해할 수 있게, 레슨 06 브라우저 자동화 입문에서는 오류 메시지을/를 검토한다는 과정을 별도의 단계로 둔다. 이때 학생은 화면에 보이는 값 하나보다 어떤 입력에서 어떤 변환을 거쳐 결과가 나왔는지 설명해야 한다. 강사는 변수명, 출력 형태, 저장 여부를 함께 확인하며 다음 실행에서도 같은 결과가 나오는지 묻는다.

### 체크포인트 24: 결과 행 수 정리한다

재실행했을 때 같은 결과를 얻기 위해, 레슨 06 브라우저 자동화 입문에서는 결과 행 수을/를 정리한다는 과정을 별도의 단계로 둔다. 이때 학생은 화면에 보이는 값 하나보다 어떤 입력에서 어떤 변환을 거쳐 결과가 나왔는지 설명해야 한다. 강사는 변수명, 출력 형태, 저장 여부를 함께 확인하며 다음 실행에서도 같은 결과가 나오는지 묻는다.

### 체크포인트 25: 검증 기준 설명한다

다음 셀에서 재사용하기 위해, 레슨 06 브라우저 자동화 입문에서는 검증 기준을/를 설명한다는 과정을 별도의 단계로 둔다. 이때 학생은 화면에 보이는 값 하나보다 어떤 입력에서 어떤 변환을 거쳐 결과가 나왔는지 설명해야 한다. 강사는 변수명, 출력 형태, 저장 여부를 함께 확인하며 다음 실행에서도 같은 결과가 나오는지 묻는다.

### 체크포인트 26: 중간 변수 저장한다

실제 사이트 확장 전에 위험을 낮추기 위해, 레슨 06 브라우저 자동화 입문에서는 중간 변수을/를 저장한다는 과정을 별도의 단계로 둔다. 이때 학생은 화면에 보이는 값 하나보다 어떤 입력에서 어떤 변환을 거쳐 결과가 나왔는지 설명해야 한다. 강사는 변수명, 출력 형태, 저장 여부를 함께 확인하며 다음 실행에서도 같은 결과가 나오는지 묻는다.

### 체크포인트 27: 입력 파일 되돌아본다

저장 파일의 신뢰도를 높이기 위해, 레슨 06 브라우저 자동화 입문에서는 입력 파일을/를 되돌아본다는 과정을 별도의 단계로 둔다. 이때 학생은 화면에 보이는 값 하나보다 어떤 입력에서 어떤 변환을 거쳐 결과가 나왔는지 설명해야 한다. 강사는 변수명, 출력 형태, 저장 여부를 함께 확인하며 다음 실행에서도 같은 결과가 나오는지 묻는다.

### 체크포인트 28: 선택자 표준화한다

코랩과 로컬 실행 차이를 줄이기 위해, 레슨 06 브라우저 자동화 입문에서는 선택자을/를 표준화한다는 과정을 별도의 단계로 둔다. 이때 학생은 화면에 보이는 값 하나보다 어떤 입력에서 어떤 변환을 거쳐 결과가 나왔는지 설명해야 한다. 강사는 변수명, 출력 형태, 저장 여부를 함께 확인하며 다음 실행에서도 같은 결과가 나오는지 묻는다.
