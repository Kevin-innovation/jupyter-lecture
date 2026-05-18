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

# 레슨 03 — 최종 미션

[![Open in Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/Kevin-innovation/jupyter-lecture/blob/main/python-web-automation/lectures/03/%5B%ED%95%99%EC%83%9D%EC%9A%A9%5D%20%EB%A0%88%EC%8A%A8%2003%20%E2%80%94%20HTML%20%ED%85%8C%EC%9D%B4%EB%B8%94%EA%B3%BC%20%EB%A6%AC%EC%8A%A4%ED%8A%B8%20%EB%8D%B0%EC%9D%B4%ED%84%B0%20%EC%A0%95%EB%A6%AC.ipynb)

대시보드 표, 피드백 카드, todo 리스트, 루브릭 CSV를 읽어 하나의 운영 요약을 만든다. 이번 미션은 실제 사이트를 요청하지 않고 합성 fixture만 사용한다.

## 목표

- class_dashboard.html에서 학생별 records를 만든다.
- feedback_cards.html에서 high priority 제목을 모은다.
- todo_list.html에서 상태별 개수를 센다.
- rubric.csv에서 총점과 평가 항목을 읽는다.
- 결과를 CSV 또는 JSON으로 저장하고 3문장 요약을 작성한다.

## 0. 환경 셀

~~~python
import os
import re
import csv
import json
from pathlib import Path
from collections import Counter, defaultdict

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
    DATA_BASE = 'https://raw.githubusercontent.com/Kevin-innovation/jupyter-lecture/main/python-web-automation/lectures/03/data'
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

def load_bytes(filename):
    if DATA_BASE.startswith('http'):
        url = f'{DATA_BASE}/{filename}'
        response = requests.get(url, timeout=10, headers={'User-Agent': 'D-Lab-Lesson/1.0'})
        response.raise_for_status()
        return response.content
    return Path(DATA_BASE, filename).read_bytes()

def read_soup(filename):
    return BeautifulSoup(load_text(filename), 'html.parser')

def clean_int(text):
    return int(re.sub(r'[^0-9]', '', str(text)))

def percent_to_int(text):
    return clean_int(text)

def safe_text(node, default=''):
    return node.text.strip() if node else default

print('colab:', IS_COLAB)
print('data base:', DATA_BASE)
~~~

## 제출 산출물

1. 실행 가능한 노트북
2. lesson03_final_summary.csv 또는 lesson03_final_summary.json
3. 자동화 결과 요약 3문장
4. 안전 규칙 점검 메모 2개

## 구현 조건

- 반복 단위 selector를 코드에 명확히 드러낸다.
- progress, submissions, passed, minutes, max_score는 숫자로 변환한다.
- 저장 파일에는 최소 course, student_count, avg_progress, watch_count 중 세 가지 이상을 포함한다.
- high priority 피드백 수와 pending todo 수를 요약에 포함한다.
- 실제 사이트로 확장할 때 지켜야 할 안전 규칙을 주석이나 메모로 남긴다.

## 스타터 코드

~~~python
# 1. table records 만들기
summary_rows = []

# 2. card/list/csv 요약 만들기
operation_note = {}

# 3. CSV 또는 JSON으로 저장하기
# TODO
~~~

## 자동화 결과 요약 양식

- 수집 대상:
- 핵심 결과:
- 다음 실행 때 조심할 점:

## 안전 규칙 메모 양식

- 요청 간격 또는 최대 요청 수:
- 개인정보와 약관 확인:

## 제출 전 체크

- 같은 셀을 다시 실행해도 결과 파일이 정상적으로 덮어써지는가.
- 저장 파일을 열었을 때 컬럼 이름이 운영자가 이해할 수 있는가.
- 빈 결과가 나왔을 때 성공으로 오해하지 않도록 개수 출력이 있는가.
- fixture 이름이 코드에 정확히 들어갔는가.

## 평가 루브릭

| 항목 | 배점 | 확인 방법 |
|---|---:|---|
| selector 정확도 | 30 | table, card, list 반복 단위가 정확하다 |
| 타입 변환 | 20 | progress, minutes, max_score가 숫자로 변환된다 |
| 요약 품질 | 20 | 코스별 평균과 상태별 개수를 포함한다 |
| 산출물 저장 | 20 | CSV 또는 JSON 파일을 재실행 가능하게 만든다 |
| 안전 메모 | 10 | 실제 사이트 확장 전 확인 기준을 적는다 |

## 미션 운영 팁

최종 미션은 문제 1~15를 그대로 복사하는 과제가 아니다. 필요한 코드를 함수로 묶거나 순서를 다시 정리해도 된다. 다만 결과 파일에는 운영자가 읽을 수 있는 컬럼 이름이 있어야 하고, 요약 문장에는 수집 대상과 핵심 결과가 들어가야 한다.

제출 전에는 런타임을 다시 시작한 뒤 환경 셀부터 마지막 저장 셀까지 한 번에 실행한다. 중간 변수에 의존해 우연히 동작하는 코드는 실제 운영 자동화로 볼 수 없다.

