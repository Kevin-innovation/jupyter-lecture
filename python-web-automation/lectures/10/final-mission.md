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

# 레슨 10 — 최종 미션

[![Open in Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/Kevin-innovation/jupyter-lecture/blob/main/python-web-automation/lectures/10/%5B%ED%95%99%EC%83%9D%EC%9A%A9%5D%20%EB%A0%88%EC%8A%A8%2010%20%E2%80%94%20%ED%86%B5%ED%95%A9%20%ED%94%84%EB%A1%9C%EC%A0%9D%ED%8A%B8%3A%20%EC%9E%90%EB%8F%99%ED%99%94%20%EB%A6%AC%ED%8F%AC%ED%8A%B8%20%EB%A7%8C%EB%93%A4%EA%B8%B0.ipynb)

> 코랩에서 최종 미션을 진행하려면 우측 상단 또는 아래의 **Open in Colab** 버튼을 클릭한다.


## 프로젝트: 자동화 포털 운영 리포트

이번 레슨에서 배운 내용을 하나의 작은 운영 자동화로 묶는다. 제공된 fixture만 사용하고 외부 사이트에는 요청하지 않는다.

## 요구사항

1. 레슨의 주요 입력 파일을 모두 읽는다.
2. 최소 2개 이상의 검증 기준을 적용한다.
3. 중복, 오류, 성공 건수를 분리해서 계산한다.
4. 결과 CSV 또는 JSON 중 하나 이상을 저장한다.
5. 마지막에 운영자가 읽을 수 있는 3문장 요약을 작성한다.

## 보너스

- SQLite 저장 또는 상태 코드별 요약을 추가한다.
- 오류가 있는 항목만 따로 저장한다.

## 시작 코드

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
    DATA_BASE = 'https://raw.githubusercontent.com/Kevin-innovation/jupyter-lecture/main/python-web-automation/lectures/10/data'
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

def parse_html(filename):
    return BeautifulSoup(load_text(filename), 'html.parser')

def text_or_empty(el):
    return '' if el is None else el.get_text(' ', strip=True)

def status_ok(status):
    return str(status).strip().lower() in {'open', 'ready', 'published'}

def make_key(*parts):
    return '::'.join(str(part).strip().lower() for part in parts)
~~~

## 자동화 결과 요약

- 오늘 확인한 입력:
- 저장한 산출물:
- 다음에 개선할 점:

## 제출 전 자기 점검

- 포털 시작 페이지, 공지, 과정, 다운로드 manifest, 상태 metric, 품질 규칙을 모두 읽었다.
- 리포트 대상 과정을 고르는 기준을 코드에 주석이나 변수명으로 드러냈다.
- 중복 키 또는 자료 개수 검증처럼 저장 전 품질 확인을 최소 1개 이상 넣었다.
- 저장 파일 이름과 행 수를 마지막 출력에서 확인했다.
- 운영 메모 3문장에는 구체적인 숫자와 다음 실행 시 조심할 점이 들어 있다.

## 발표 가이드

발표는 1분 안에 끝낸다. 첫 문장은 “어떤 입력을 읽었는지”, 두 번째 문장은 “어떤 기준으로 리포트 대상을 골랐는지”, 세 번째 문장은 “어떤 파일을 저장했고 다음에 무엇을 개선할지”로 구성한다. 코드 전체를 설명하지 말고 자동화 흐름과 산출물 중심으로 말한다.


## 산출물 예시

최종 제출물은 노트북 실행 결과만으로 끝나지 않는다. 아래 중 최소 2개 이상을 남긴다.

- lesson10_portal_report.csv: 리포트 대상 과정별 상세 행
- lesson10_portal_report.json: 전체 입력과 리포트 대상 개수 요약
- lesson10_portal.db: report 테이블로 저장한 조회용 데이터
- 운영 메모 3문장: 다음 담당자가 읽을 요약

## 감점 기준

- 외부 사이트에 직접 요청을 보내면 감점한다.
- fixture 파일명을 하드코딩하더라도 입력 구조를 확인하지 않으면 감점한다.
- CSV나 JSON 저장 없이 print만 있으면 최종 미션 완료로 보지 않는다.
- 운영 메모에 숫자가 없으면 리포트로 보기 어렵다.
