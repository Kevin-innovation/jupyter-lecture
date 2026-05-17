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

# 레슨 07 — 안정적인 selector와 wait

이 노트북은 읽기와 따라하기용 강의 노트북이다. 학생은 셀을 위에서 아래로 실행하며 웹 자동화에서 상태가 어떻게 유지되고 화면 조작이 어떤 순서로 기록되는지 확인한다. 안정적인 selector와 wait는 실제 사이트 대신 합성 fixture로 안전하게 연습한다.

## 학습 목표

1. 불안정한 class selector와 안정적인 data-testid selector를 구분한다.
2. 요소가 늦게 나타나는 상황을 wait로 처리한다.
3. timeout 실패를 명확히 기록한다.
4. wait 케이스를 CSV 기반으로 반복 검증한다.
5. selector 추천표와 결과 로그를 저장한다.

---

## 1. 안정 selector 기준

자동화에서는 화면 문구와 임의 class보다 data-testid, role, 고유 id가 안정적이다.

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

lab = MiniPage(load_text('selector_lab.html'))
print(lab.locator('[data-testid="save-button"]').text_content())
~~~

---

## 2. wait가 필요한 이유

실제 화면은 네트워크와 렌더링 때문에 늦게 나타나는 요소가 있다. 조건이 맞을 때까지 기다리는 방식이 sleep보다 안정적이다.

~~~python
page = MiniPage(load_text('dynamic_dashboard.html'))
el = page.wait_for_selector('[data-testid="metric-submit"]', timeout_steps=2)
print(page.step, el.get_text(' ', strip=True))
~~~

---

## 3. 실패도 기록하기

없는 요소를 기다릴 때는 TimeoutError가 나야 잘못된 selector를 빨리 수정할 수 있다.

~~~python
try:
    page.wait_for_selector('[data-testid="missing"]', timeout_steps=1)
except Exception as error:
    print(type(error).__name__)
~~~

---

## 데이터 출처와 안전 규칙

이 레슨의 파일은 모두 수업용 합성 데이터다. 실제 사이트의 개인정보, 로그인 정보, 유료 콘텐츠를 포함하지 않는다. 실제 사이트로 확장할 때는 robots.txt, 이용 약관, 요청 간격, 개인정보 여부를 먼저 확인한다. 수업 중에는 fixture를 반복 실행하며 구조를 익히고, 외부 사이트를 빠르게 반복 요청하지 않는다.

---

## 강의 보강 노트

이 절은 수업 중 교사가 질문으로 풀어낼 수 있는 운영형 설명이다. 학생이 셀을 실행한 뒤 결과만 맞히지 않고 자동화 절차를 말로 설명하도록 돕는다.

### 1. 안정 selector의 우선순위

자동화 selector는 예쁘게 보이는 class보다 의미가 고정된 data-testid, role, label, id를 우선한다. 학생에게 class가 짧고 쉬워 보여도 배포 때 바뀔 수 있다는 점을 실제 사례로 설명한다.

### 2. wait와 sleep의 차이

sleep은 시간을 기다리고 wait는 조건을 기다린다. 화면이 빨리 뜨면 sleep은 시간을 낭비하고, 늦게 뜨면 실패할 수 있으므로 조건 기반 wait가 유지보수에 유리하다.

### 3. timeout을 숨기지 않기

요소가 끝까지 나타나지 않으면 명확한 TimeoutError가 나야 한다. 실패를 빈 문자열로 바꾸면 이후 단계에서 원인을 잃어버리므로 학생 답안에는 실패 메시지가 남아야 한다.

### 4. 동적 DOM 읽기

동적 화면은 처음 HTML에 모든 값이 있는 것이 아니다. 이 레슨의 fixture는 step이 진행되며 보이는 요소가 달라지도록 만들어, 학생이 기다림의 필요성을 눈으로 확인하게 한다.

### 5. selector 추천표

운영 코드에는 어떤 selector를 선택했는지 이유를 남기면 좋다. data-testid를 사용한 이유, 대체 selector, 피해야 할 selector를 표로 정리하면 다음 수정자가 빠르게 이해한다.

### 6. 테스트 케이스 반복

wait_cases.csv처럼 조건을 표로 만들면 여러 selector와 timeout 값을 반복 검증할 수 있다. 자동화 스크립트가 커질수록 이런 표 기반 점검이 수동 클릭보다 빠르다.

### 7. 수업 피드백

학생이 실패하면 timeout을 무작정 늘리기보다 selector가 맞는지, hidden 상태인지, step이 증가했는지 순서대로 확인하게 한다. 이 순서가 실제 브라우저 자동화 디버깅과 같다.

### 체크포인트 1: 운영 메모 저장한다

코랩과 로컬 실행 차이를 줄이기 위해, 레슨 07 안정적인 selector와 wait에서는 운영 메모을/를 저장한다는 과정을 별도의 단계로 둔다. 이때 학생은 화면에 보이는 값 하나보다 어떤 입력에서 어떤 변환을 거쳐 결과가 나왔는지 설명해야 한다. 강사는 변수명, 출력 형태, 저장 여부를 함께 확인하며 다음 실행에서도 같은 결과가 나오는지 묻는다.

### 체크포인트 2: 출력 형태 되돌아본다

수업 중 피드백 시간을 줄이기 위해, 레슨 07 안정적인 selector와 wait에서는 출력 형태을/를 되돌아본다는 과정을 별도의 단계로 둔다. 이때 학생은 화면에 보이는 값 하나보다 어떤 입력에서 어떤 변환을 거쳐 결과가 나왔는지 설명해야 한다. 강사는 변수명, 출력 형태, 저장 여부를 함께 확인하며 다음 실행에서도 같은 결과가 나오는지 묻는다.

### 체크포인트 3: 반복 단위 표준화한다

학생이 막힌 지점을 찾기 위해, 레슨 07 안정적인 selector와 wait에서는 반복 단위을/를 표준화한다는 과정을 별도의 단계로 둔다. 이때 학생은 화면에 보이는 값 하나보다 어떤 입력에서 어떤 변환을 거쳐 결과가 나왔는지 설명해야 한다. 강사는 변수명, 출력 형태, 저장 여부를 함께 확인하며 다음 실행에서도 같은 결과가 나오는지 묻는다.

### 체크포인트 4: 상태 값 검증한다

운영자가 결과를 이해할 수 있게, 레슨 07 안정적인 selector와 wait에서는 상태 값을/를 검증한다는 과정을 별도의 단계로 둔다. 이때 학생은 화면에 보이는 값 하나보다 어떤 입력에서 어떤 변환을 거쳐 결과가 나왔는지 설명해야 한다. 강사는 변수명, 출력 형태, 저장 여부를 함께 확인하며 다음 실행에서도 같은 결과가 나오는지 묻는다.

### 체크포인트 5: 저장 경로 확인한다

재실행했을 때 같은 결과를 얻기 위해, 레슨 07 안정적인 selector와 wait에서는 저장 경로을/를 확인한다는 과정을 별도의 단계로 둔다. 이때 학생은 화면에 보이는 값 하나보다 어떤 입력에서 어떤 변환을 거쳐 결과가 나왔는지 설명해야 한다. 강사는 변수명, 출력 형태, 저장 여부를 함께 확인하며 다음 실행에서도 같은 결과가 나오는지 묻는다.

### 체크포인트 6: 실행 순서 분리한다

다음 셀에서 재사용하기 위해, 레슨 07 안정적인 selector와 wait에서는 실행 순서을/를 분리한다는 과정을 별도의 단계로 둔다. 이때 학생은 화면에 보이는 값 하나보다 어떤 입력에서 어떤 변환을 거쳐 결과가 나왔는지 설명해야 한다. 강사는 변수명, 출력 형태, 저장 여부를 함께 확인하며 다음 실행에서도 같은 결과가 나오는지 묻는다.

### 체크포인트 7: 운영 메모 기록한다

실제 사이트 확장 전에 위험을 낮추기 위해, 레슨 07 안정적인 selector와 wait에서는 운영 메모을/를 기록한다는 과정을 별도의 단계로 둔다. 이때 학생은 화면에 보이는 값 하나보다 어떤 입력에서 어떤 변환을 거쳐 결과가 나왔는지 설명해야 한다. 강사는 변수명, 출력 형태, 저장 여부를 함께 확인하며 다음 실행에서도 같은 결과가 나오는지 묻는다.

### 체크포인트 8: 출력 형태 비교한다

저장 파일의 신뢰도를 높이기 위해, 레슨 07 안정적인 selector와 wait에서는 출력 형태을/를 비교한다는 과정을 별도의 단계로 둔다. 이때 학생은 화면에 보이는 값 하나보다 어떤 입력에서 어떤 변환을 거쳐 결과가 나왔는지 설명해야 한다. 강사는 변수명, 출력 형태, 저장 여부를 함께 확인하며 다음 실행에서도 같은 결과가 나오는지 묻는다.

### 체크포인트 9: 반복 단위 요약한다

코랩과 로컬 실행 차이를 줄이기 위해, 레슨 07 안정적인 selector와 wait에서는 반복 단위을/를 요약한다는 과정을 별도의 단계로 둔다. 이때 학생은 화면에 보이는 값 하나보다 어떤 입력에서 어떤 변환을 거쳐 결과가 나왔는지 설명해야 한다. 강사는 변수명, 출력 형태, 저장 여부를 함께 확인하며 다음 실행에서도 같은 결과가 나오는지 묻는다.

### 체크포인트 10: 상태 값 검토한다

수업 중 피드백 시간을 줄이기 위해, 레슨 07 안정적인 selector와 wait에서는 상태 값을/를 검토한다는 과정을 별도의 단계로 둔다. 이때 학생은 화면에 보이는 값 하나보다 어떤 입력에서 어떤 변환을 거쳐 결과가 나왔는지 설명해야 한다. 강사는 변수명, 출력 형태, 저장 여부를 함께 확인하며 다음 실행에서도 같은 결과가 나오는지 묻는다.

### 체크포인트 11: 저장 경로 정리한다

학생이 막힌 지점을 찾기 위해, 레슨 07 안정적인 selector와 wait에서는 저장 경로을/를 정리한다는 과정을 별도의 단계로 둔다. 이때 학생은 화면에 보이는 값 하나보다 어떤 입력에서 어떤 변환을 거쳐 결과가 나왔는지 설명해야 한다. 강사는 변수명, 출력 형태, 저장 여부를 함께 확인하며 다음 실행에서도 같은 결과가 나오는지 묻는다.

### 체크포인트 12: 실행 순서 설명한다

운영자가 결과를 이해할 수 있게, 레슨 07 안정적인 selector와 wait에서는 실행 순서을/를 설명한다는 과정을 별도의 단계로 둔다. 이때 학생은 화면에 보이는 값 하나보다 어떤 입력에서 어떤 변환을 거쳐 결과가 나왔는지 설명해야 한다. 강사는 변수명, 출력 형태, 저장 여부를 함께 확인하며 다음 실행에서도 같은 결과가 나오는지 묻는다.

### 체크포인트 13: 운영 메모 저장한다

재실행했을 때 같은 결과를 얻기 위해, 레슨 07 안정적인 selector와 wait에서는 운영 메모을/를 저장한다는 과정을 별도의 단계로 둔다. 이때 학생은 화면에 보이는 값 하나보다 어떤 입력에서 어떤 변환을 거쳐 결과가 나왔는지 설명해야 한다. 강사는 변수명, 출력 형태, 저장 여부를 함께 확인하며 다음 실행에서도 같은 결과가 나오는지 묻는다.

### 체크포인트 14: 출력 형태 되돌아본다

다음 셀에서 재사용하기 위해, 레슨 07 안정적인 selector와 wait에서는 출력 형태을/를 되돌아본다는 과정을 별도의 단계로 둔다. 이때 학생은 화면에 보이는 값 하나보다 어떤 입력에서 어떤 변환을 거쳐 결과가 나왔는지 설명해야 한다. 강사는 변수명, 출력 형태, 저장 여부를 함께 확인하며 다음 실행에서도 같은 결과가 나오는지 묻는다.

### 체크포인트 15: 반복 단위 표준화한다

실제 사이트 확장 전에 위험을 낮추기 위해, 레슨 07 안정적인 selector와 wait에서는 반복 단위을/를 표준화한다는 과정을 별도의 단계로 둔다. 이때 학생은 화면에 보이는 값 하나보다 어떤 입력에서 어떤 변환을 거쳐 결과가 나왔는지 설명해야 한다. 강사는 변수명, 출력 형태, 저장 여부를 함께 확인하며 다음 실행에서도 같은 결과가 나오는지 묻는다.

### 체크포인트 16: 상태 값 검증한다

저장 파일의 신뢰도를 높이기 위해, 레슨 07 안정적인 selector와 wait에서는 상태 값을/를 검증한다는 과정을 별도의 단계로 둔다. 이때 학생은 화면에 보이는 값 하나보다 어떤 입력에서 어떤 변환을 거쳐 결과가 나왔는지 설명해야 한다. 강사는 변수명, 출력 형태, 저장 여부를 함께 확인하며 다음 실행에서도 같은 결과가 나오는지 묻는다.

### 체크포인트 17: 저장 경로 확인한다

코랩과 로컬 실행 차이를 줄이기 위해, 레슨 07 안정적인 selector와 wait에서는 저장 경로을/를 확인한다는 과정을 별도의 단계로 둔다. 이때 학생은 화면에 보이는 값 하나보다 어떤 입력에서 어떤 변환을 거쳐 결과가 나왔는지 설명해야 한다. 강사는 변수명, 출력 형태, 저장 여부를 함께 확인하며 다음 실행에서도 같은 결과가 나오는지 묻는다.

### 체크포인트 18: 실행 순서 분리한다

수업 중 피드백 시간을 줄이기 위해, 레슨 07 안정적인 selector와 wait에서는 실행 순서을/를 분리한다는 과정을 별도의 단계로 둔다. 이때 학생은 화면에 보이는 값 하나보다 어떤 입력에서 어떤 변환을 거쳐 결과가 나왔는지 설명해야 한다. 강사는 변수명, 출력 형태, 저장 여부를 함께 확인하며 다음 실행에서도 같은 결과가 나오는지 묻는다.

### 체크포인트 19: 운영 메모 기록한다

학생이 막힌 지점을 찾기 위해, 레슨 07 안정적인 selector와 wait에서는 운영 메모을/를 기록한다는 과정을 별도의 단계로 둔다. 이때 학생은 화면에 보이는 값 하나보다 어떤 입력에서 어떤 변환을 거쳐 결과가 나왔는지 설명해야 한다. 강사는 변수명, 출력 형태, 저장 여부를 함께 확인하며 다음 실행에서도 같은 결과가 나오는지 묻는다.

### 체크포인트 20: 출력 형태 비교한다

운영자가 결과를 이해할 수 있게, 레슨 07 안정적인 selector와 wait에서는 출력 형태을/를 비교한다는 과정을 별도의 단계로 둔다. 이때 학생은 화면에 보이는 값 하나보다 어떤 입력에서 어떤 변환을 거쳐 결과가 나왔는지 설명해야 한다. 강사는 변수명, 출력 형태, 저장 여부를 함께 확인하며 다음 실행에서도 같은 결과가 나오는지 묻는다.

### 체크포인트 21: 반복 단위 요약한다

재실행했을 때 같은 결과를 얻기 위해, 레슨 07 안정적인 selector와 wait에서는 반복 단위을/를 요약한다는 과정을 별도의 단계로 둔다. 이때 학생은 화면에 보이는 값 하나보다 어떤 입력에서 어떤 변환을 거쳐 결과가 나왔는지 설명해야 한다. 강사는 변수명, 출력 형태, 저장 여부를 함께 확인하며 다음 실행에서도 같은 결과가 나오는지 묻는다.

### 체크포인트 22: 상태 값 검토한다

다음 셀에서 재사용하기 위해, 레슨 07 안정적인 selector와 wait에서는 상태 값을/를 검토한다는 과정을 별도의 단계로 둔다. 이때 학생은 화면에 보이는 값 하나보다 어떤 입력에서 어떤 변환을 거쳐 결과가 나왔는지 설명해야 한다. 강사는 변수명, 출력 형태, 저장 여부를 함께 확인하며 다음 실행에서도 같은 결과가 나오는지 묻는다.

### 체크포인트 23: 저장 경로 정리한다

실제 사이트 확장 전에 위험을 낮추기 위해, 레슨 07 안정적인 selector와 wait에서는 저장 경로을/를 정리한다는 과정을 별도의 단계로 둔다. 이때 학생은 화면에 보이는 값 하나보다 어떤 입력에서 어떤 변환을 거쳐 결과가 나왔는지 설명해야 한다. 강사는 변수명, 출력 형태, 저장 여부를 함께 확인하며 다음 실행에서도 같은 결과가 나오는지 묻는다.

### 체크포인트 24: 실행 순서 설명한다

저장 파일의 신뢰도를 높이기 위해, 레슨 07 안정적인 selector와 wait에서는 실행 순서을/를 설명한다는 과정을 별도의 단계로 둔다. 이때 학생은 화면에 보이는 값 하나보다 어떤 입력에서 어떤 변환을 거쳐 결과가 나왔는지 설명해야 한다. 강사는 변수명, 출력 형태, 저장 여부를 함께 확인하며 다음 실행에서도 같은 결과가 나오는지 묻는다.

### 체크포인트 25: 운영 메모 저장한다

코랩과 로컬 실행 차이를 줄이기 위해, 레슨 07 안정적인 selector와 wait에서는 운영 메모을/를 저장한다는 과정을 별도의 단계로 둔다. 이때 학생은 화면에 보이는 값 하나보다 어떤 입력에서 어떤 변환을 거쳐 결과가 나왔는지 설명해야 한다. 강사는 변수명, 출력 형태, 저장 여부를 함께 확인하며 다음 실행에서도 같은 결과가 나오는지 묻는다.

### 체크포인트 26: 출력 형태 되돌아본다

수업 중 피드백 시간을 줄이기 위해, 레슨 07 안정적인 selector와 wait에서는 출력 형태을/를 되돌아본다는 과정을 별도의 단계로 둔다. 이때 학생은 화면에 보이는 값 하나보다 어떤 입력에서 어떤 변환을 거쳐 결과가 나왔는지 설명해야 한다. 강사는 변수명, 출력 형태, 저장 여부를 함께 확인하며 다음 실행에서도 같은 결과가 나오는지 묻는다.

### 체크포인트 27: 반복 단위 표준화한다

학생이 막힌 지점을 찾기 위해, 레슨 07 안정적인 selector와 wait에서는 반복 단위을/를 표준화한다는 과정을 별도의 단계로 둔다. 이때 학생은 화면에 보이는 값 하나보다 어떤 입력에서 어떤 변환을 거쳐 결과가 나왔는지 설명해야 한다. 강사는 변수명, 출력 형태, 저장 여부를 함께 확인하며 다음 실행에서도 같은 결과가 나오는지 묻는다.

### 체크포인트 28: 상태 값 검증한다

운영자가 결과를 이해할 수 있게, 레슨 07 안정적인 selector와 wait에서는 상태 값을/를 검증한다는 과정을 별도의 단계로 둔다. 이때 학생은 화면에 보이는 값 하나보다 어떤 입력에서 어떤 변환을 거쳐 결과가 나왔는지 설명해야 한다. 강사는 변수명, 출력 형태, 저장 여부를 함께 확인하며 다음 실행에서도 같은 결과가 나오는지 묻는다.

### 체크포인트 29: 저장 경로 확인한다

재실행했을 때 같은 결과를 얻기 위해, 레슨 07 안정적인 selector와 wait에서는 저장 경로을/를 확인한다는 과정을 별도의 단계로 둔다. 이때 학생은 화면에 보이는 값 하나보다 어떤 입력에서 어떤 변환을 거쳐 결과가 나왔는지 설명해야 한다. 강사는 변수명, 출력 형태, 저장 여부를 함께 확인하며 다음 실행에서도 같은 결과가 나오는지 묻는다.
