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

# 레슨 04 — 최종 미션

[![Open in Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/Kevin-innovation/jupyter-lecture/blob/main/python-web-automation/lectures/04/%5B%ED%95%99%EC%83%9D%EC%9A%A9%5D%20%EB%A0%88%EC%8A%A8%2004%20%E2%80%94%20%ED%8C%8C%EC%9D%BC%20%EB%8B%A4%EC%9A%B4%EB%A1%9C%EB%93%9C%EC%99%80%20%ED%8F%B4%EB%8D%94%20%EC%A0%95%EB%A6%AC.ipynb)

자료실 HTML과 manifest.csv를 기준으로 파일을 저장하고, 저장 로그와 요약 파일을 만든다. 이번 미션은 실제 사이트 요청 없이 합성 fixture만 사용한다.

## 목표

- resource_center.html에서 a.file-link 목록을 읽는다.
- manifest.csv를 기준으로 다운로드 대상 목록을 정리한다.
- group 또는 week 기준으로 폴더를 나누어 저장한다.
- 저장된 파일의 크기를 검증한다.
- download_log.csv와 download_summary.json을 만든다.

## 0. 환경 셀

~~~python
import os
import re
import csv
import json
import shutil
from pathlib import Path
from collections import Counter
from urllib.parse import urljoin

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
    DATA_BASE = 'https://raw.githubusercontent.com/Kevin-innovation/jupyter-lecture/main/python-web-automation/lectures/04/data'
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

def safe_filename(name):
    base = Path(str(name)).name
    cleaned = re.sub(r'[^0-9A-Za-z가-힣_.-]+', '_', base).strip('._')
    return cleaned or 'downloaded_file'

def save_bytes(relative_path, target_dir):
    target_dir = Path(target_dir)
    target_dir.mkdir(parents=True, exist_ok=True)
    filename = safe_filename(relative_path)
    target = target_dir / filename
    target.write_bytes(load_bytes(relative_path))
    return target

def file_info(path):
    path = Path(path)
    return {'path': str(path), 'name': path.name, 'size': path.stat().st_size, 'exists': path.exists()}

print('colab:', IS_COLAB)
print('data base:', DATA_BASE)
~~~

## 제출 산출물

1. 실행 가능한 노트북
2. downloads/lesson04/download_log.csv
3. downloads/lesson04/download_summary.json
4. 자동화 결과 요약 3문장
5. 안전 규칙 점검 메모 2개

## 구현 조건

- 저장 기준은 manifest.csv를 우선으로 한다.
- 폴더는 group 또는 week 기준으로 나누되 코드에 기준을 명시한다.
- 파일 저장 후 size가 0인 파일을 검출한다.
- CSV 로그에는 file_name, group, type, saved_path, size, status를 포함한다.
- JSON 요약에는 전체 파일 수, type별 개수, empty 파일 수를 포함한다.

## 스타터 코드

~~~python
# 1. manifest 읽기
manifest = []

# 2. 파일 저장과 로그 생성
download_log = []

# 3. CSV/JSON 산출물 저장
# TODO
~~~

## 자동화 결과 요약 양식

- 수집 대상:
- 핵심 결과:
- 다음 실행 때 조심할 점:

## 안전 규칙 메모 양식

- 요청 간격 또는 최대 파일 수:
- 저장 위치와 개인정보 확인:

## 평가 루브릭

| 항목 | 배점 | 확인 방법 |
|---|---:|---|
| 대상 식별 | 25 | HTML 링크와 manifest 경로를 정확히 읽는다 |
| 저장 구조 | 25 | group 또는 week 기준으로 폴더를 나눈다 |
| 검증 품질 | 20 | size와 행 수를 확인한다 |
| 산출물 | 20 | CSV 로그와 JSON 요약을 만든다 |
| 안전 메모 | 10 | 실제 사이트 확장 기준을 적는다 |

## 제출 전 체크

- 새 런타임에서 전체 실행해도 같은 산출물이 만들어지는가.
- 빈 파일이 없는지 확인했는가.
- 로그 행 수가 manifest 행 수와 같은가.
- 실제 사이트가 아닌 fixture만 사용했는가.

## 운영 기준

최종 미션에서 산출물은 실행 결과만 보여주는 것이 아니라 다음 실행의 기준이 되어야 한다. download_log.csv는 어떤 파일을 어디에 저장했는지 확인하는 장부이고, download_summary.json은 전체 상태를 빠르게 보는 요약이다. 두 파일 중 하나라도 빠지면 운영 자동화 완성으로 보지 않는다.

실제 사이트에 적용할 때는 max_files, max_total_mb, request_interval_seconds 같은 제한값을 먼저 정한다. 이번 fixture에서는 파일이 작지만, 제한값을 생각하는 습관이 있어야 대용량 자료실에서도 사고를 줄일 수 있다.

