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

# 레슨 04 — 최종 미션 모범 답안

> 🔒 교사용. 학생에게는 최종 미션 문제 파일만 공유한다.

자료실 HTML과 manifest.csv를 기준으로 수업 자료를 그룹별 폴더에 저장하고 다운로드 로그를 만든다.

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

def clean_int(text):
    return int(re.sub(r'[^0-9]', '', str(text)))

def safe_filename(name):
    return re.sub(r'[^A-Za-z0-9_.-]+', '_', str(name)).strip('_')

def save_bytes(relative_path, target_dir):
    target_dir = Path(target_dir)
    target_dir.mkdir(parents=True, exist_ok=True)
    filename = safe_filename(Path(relative_path).name)
    target = target_dir / filename
    target.write_bytes(load_bytes(relative_path))
    return target

print('colab:', IS_COLAB)
print('data base:', DATA_BASE)
~~~

## 모범 답안

~~~python
manifest = list(csv.DictReader(load_text('manifest.csv').splitlines()))
logs = []
for row in manifest:
    path = save_bytes(row['path'], Path('downloads/lesson04/final') / row['group'])
    logs.append({'file_name': row['file_name'], 'group': row['group'], 'saved_path': str(path), 'size': path.stat().st_size})
with open('lesson04_download_log.csv', 'w', newline='', encoding='utf-8') as f:
    writer = csv.DictWriter(f, fieldnames=['file_name','group','saved_path','size'])
    writer.writeheader(); writer.writerows(logs)
print('logs:', len(logs))
print('total bytes:', sum(row['size'] for row in logs))
~~~

## 채점 메모

- 코드가 한 번 실행되어 산출물을 만들고, 다시 실행해도 같은 결과가 나와야 한다.
- 결과 저장 경로, 수집 개수, 필터 조건이 요약 문장과 충돌하지 않아야 한다.
- 실제 사이트로 옮길 때 필요한 요청 간격과 오류 처리 언급이 있어야 한다.

### 보강 설명 1

최종 미션 정답은 실행 결과보다 과정 추적이 중요하다. 입력 파일, 반복 단위, selector 또는 경로, 변환 규칙을 분리하면 오류 위치를 빠르게 찾을 수 있다.

### 보강 설명 2

수업 중에는 학생에게 완성 코드를 먼저 보여주지 말고 HTML 구조나 CSV 헤더를 읽게 한다. 구조를 말로 설명할 수 있으면 코드는 짧아진다.

### 보강 설명 3

운영형 자동화는 다음 주에도 다시 실행되어야 한다. 그래서 파일명, URL, 저장 경로, 로그, 요약 문장을 코드 안에서 일관되게 남긴다.

### 보강 설명 4

실제 사이트로 확장할 때는 약관, robots.txt, 요청 간격, 개인정보 여부를 먼저 확인한다. 이 코스의 초반 fixture는 그 안전 습관을 만들기 위한 장치다.

### 보강 설명 5

최종 미션 정답은 실행 결과보다 과정 추적이 중요하다. 입력 파일, 반복 단위, selector 또는 경로, 변환 규칙을 분리하면 오류 위치를 빠르게 찾을 수 있다.

### 보강 설명 6

수업 중에는 학생에게 완성 코드를 먼저 보여주지 말고 HTML 구조나 CSV 헤더를 읽게 한다. 구조를 말로 설명할 수 있으면 코드는 짧아진다.

### 보강 설명 7

운영형 자동화는 다음 주에도 다시 실행되어야 한다. 그래서 파일명, URL, 저장 경로, 로그, 요약 문장을 코드 안에서 일관되게 남긴다.

### 보강 설명 8

실제 사이트로 확장할 때는 약관, robots.txt, 요청 간격, 개인정보 여부를 먼저 확인한다. 이 코스의 초반 fixture는 그 안전 습관을 만들기 위한 장치다.

### 보강 설명 9

최종 미션 정답은 실행 결과보다 과정 추적이 중요하다. 입력 파일, 반복 단위, selector 또는 경로, 변환 규칙을 분리하면 오류 위치를 빠르게 찾을 수 있다.

### 보강 설명 10

수업 중에는 학생에게 완성 코드를 먼저 보여주지 말고 HTML 구조나 CSV 헤더를 읽게 한다. 구조를 말로 설명할 수 있으면 코드는 짧아진다.

### 보강 설명 11

운영형 자동화는 다음 주에도 다시 실행되어야 한다. 그래서 파일명, URL, 저장 경로, 로그, 요약 문장을 코드 안에서 일관되게 남긴다.
