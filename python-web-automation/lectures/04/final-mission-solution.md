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

[![Open in Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/Kevin-innovation/jupyter-lecture/blob/main/python-web-automation/lectures/04/%EB%A0%88%EC%8A%A8%2004%20%E2%80%94%20%ED%8C%8C%EC%9D%BC%20%EB%8B%A4%EC%9A%B4%EB%A1%9C%EB%93%9C%EC%99%80%20%ED%8F%B4%EB%8D%94%20%EC%A0%95%EB%A6%AC.ipynb)

자료실 HTML과 manifest.csv를 기준으로 파일을 저장하고, 저장 로그와 요약 파일을 만든다.

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

## 모범 답안

~~~python
output_root = Path('downloads/lesson04/final')
if output_root.exists():
    shutil.rmtree(output_root)
output_root.mkdir(parents=True, exist_ok=True)

resource_soup = read_soup('resource_center.html')
links = resource_soup.select('a.file-link')
manifest = list(csv.DictReader(load_text('manifest.csv').splitlines()))
manifest_paths = {row['path'] for row in manifest}
link_paths = {link['href'] for link in links}
missing_in_html = sorted(manifest_paths - link_paths)

saved_rows = []
for row in manifest:
    target_dir = output_root / row['group']
    target = save_bytes(row['path'], target_dir)
    size = target.stat().st_size
    saved_rows.append({
        'file_name': row['file_name'],
        'group': row['group'],
        'type': row['type'],
        'week': row['week'],
        'source_path': row['path'],
        'saved_path': str(target),
        'size': size,
        'status': 'saved' if size > 0 else 'empty',
    })

log_path = output_root / 'download_log.csv'
with log_path.open('w', newline='', encoding='utf-8') as f:
    writer = csv.DictWriter(f, fieldnames=['file_name', 'group', 'type', 'week', 'source_path', 'saved_path', 'size', 'status'])
    writer.writeheader()
    writer.writerows(saved_rows)

summary = {
    'html_links': len(links),
    'manifest_rows': len(manifest),
    'saved_files': len(saved_rows),
    'empty_files': sum(1 for row in saved_rows if row['status'] == 'empty'),
    'type_counts': dict(Counter(row['type'] for row in manifest)),
    'group_counts': dict(Counter(row['group'] for row in manifest)),
    'missing_in_html': missing_in_html,
}
summary_path = output_root / 'download_summary.json'
summary_path.write_text(json.dumps(summary, ensure_ascii=False, indent=2), encoding='utf-8')
print('log:', log_path, len(saved_rows))
print('summary:', summary)
~~~

## 왜 이 풀이가 기준 답안인지

manifest를 기준으로 다운로드 대상을 정하고, HTML 링크와 manifest 경로를 비교해 누락을 확인한다. 저장은 group 기준 폴더로 분리하고, 각 파일의 크기와 상태를 로그로 남긴다. CSV 로그는 행 단위 검수에 적합하고 JSON 요약은 전체 파일 수와 유형별 개수를 빠르게 확인하는 데 적합하다.

## 채점 메모

- manifest 행 수와 saved_rows 길이가 같아야 한다.
- empty_files가 0이어야 한다.
- download_log.csv에는 header와 모든 행이 있어야 한다.
- download_summary.json에는 type_counts와 group_counts가 있어야 한다.
- 실제 사이트 확장 전 최대 파일 수, 요청 간격, 저장 권한 기준을 설명할 수 있어야 한다.
