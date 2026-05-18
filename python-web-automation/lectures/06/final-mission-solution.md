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

# 레슨 06 — 최종 미션 모범 답안

[![Open in Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/Kevin-innovation/jupyter-lecture/blob/main/python-web-automation/lectures/06/%EB%A0%88%EC%8A%A8%2006%20%E2%80%94%20%EB%B8%8C%EB%9D%BC%EC%9A%B0%EC%A0%80%20%EC%9E%90%EB%8F%99%ED%99%94%20%EC%9E%85%EB%AC%B8.ipynb)

> 🔒 교사용. 학생에게는 최종 미션 문제 파일만 공유한다.

폼 입력, Todo 클릭, 탭 전환을 자동화하고 QA 로그를 만든다.

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

## 모범 답안

~~~python
cases = list(csv.DictReader(load_text('form_cases.csv').splitlines()))
qa_rows = []
for row in cases:
    p = MiniPage(load_text('form_page.html'))
    p.fill('#student-name', row['name'])
    p.fill('#course-name', row['course'])
    p.fill('#memo', row['memo'])
    p.click('#submit-profile')
    result = p.text_content('#result')
    qa_rows.append({'result': result, 'ok': ' / ' in result})
with open('lesson06_qa_log.csv', 'w', newline='', encoding='utf-8') as f:
    writer = csv.DictWriter(f, fieldnames=['result','ok'])
    writer.writeheader(); writer.writerows(qa_rows)
print('cases:', len(qa_rows))
print('passed:', sum(row['ok'] for row in qa_rows))
~~~

## 채점 메모

- 코드가 한 번 실행되어 산출물을 만들고, 다시 실행해도 같은 결과가 나와야 한다.
- 상태 변화, selector, 저장 경로, 수집 개수가 요약 문장과 충돌하지 않아야 한다.
- 실제 사이트로 옮길 때 필요한 요청 간격과 오류 처리 언급이 있어야 한다.
