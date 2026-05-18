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

# 레슨 04 — 실습 문제

[![Open in Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/Kevin-innovation/jupyter-lecture/blob/main/python-web-automation/lectures/04/%5B%ED%95%99%EC%83%9D%EC%9A%A9%5D%20%EB%A0%88%EC%8A%A8%2004%20%E2%80%94%20%ED%8C%8C%EC%9D%BC%20%EB%8B%A4%EC%9A%B4%EB%A1%9C%EB%93%9C%EC%99%80%20%ED%8F%B4%EB%8D%94%20%EC%A0%95%EB%A6%AC.ipynb)

파일 다운로드와 폴더 정리 레슨의 학생용 문제 노트북이다. 강의 노트북을 먼저 실행한 뒤 빈칸을 직접 채운다. 문제는 자료실 링크 확인에서 시작해 manifest 기반 저장, 로그 CSV, 요약 JSON까지 이어진다.

## 통과 기준

- 총 15문제 중 12문제 이상 정상 출력이면 통과.
- 문제 1~5는 HTML 링크와 manifest 구조 확인, 6~10은 파일명 정리와 저장, 11~15는 검증과 로그 저장이다.
- 정답값과 완성 코드는 적지 않는다. 파일 경로, 개수, 로그 형태를 보고 직접 판단한다.

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

---

## 문제 1 — 자료실 HTML 열기

resource_center.html을 읽고 h1 제목을 출력한다.

**기대 결과 형태**: 자료실 제목 한 줄이 출력된다.

**빈칸 힌트**: 파일은 load_text로 읽고 HTML 객체는 BeautifulSoup으로 만든다.

~~~python
html_text = ____(____)
soup = ____(html_text, 'html.parser')
print(soup.____('____').text.strip())
~~~

---

## 문제 2 — 파일 링크 개수 세기

자료실에서 다운로드 대상인 a.file-link를 모두 선택한다.

**기대 결과 형태**: links: 숫자 형태로 링크 개수가 출력된다.

**빈칸 힌트**: 반복 단위는 a 태그와 file-link class를 함께 사용한다.

~~~python
links = soup.select('____')
print('links:', ____)
~~~

---

## 문제 3 — 첫 링크 정보 읽기

첫 번째 링크에서 파일명, href, type, group을 읽는다.

**기대 결과 형태**: 파일명, 경로, 타입, 그룹이 출력된다.

**빈칸 힌트**: href와 data 속성은 태그 속성으로 읽는다.

~~~python
first = links[____]
print(first.text.strip())
print(first['____'])
print(first['____'], first['____'])
~~~

---

## 문제 4 — 소스 위치 만들기

첫 링크의 href를 현재 실행 환경에 맞는 원격 URL 또는 로컬 경로로 바꾼다.

**기대 결과 형태**: raw URL 또는 data/files 경로가 출력된다.

**빈칸 힌트**: 원격이면 urljoin, 로컬이면 Path를 사용한다.

~~~python
href = first['href']
if DATA_BASE.startswith('http'):
    print(____(DATA_BASE + '/', href))
else:
    print(Path(DATA_BASE) / href)
~~~

---

## 문제 5 — manifest 읽기

manifest.csv를 읽고 파일명과 그룹을 출력한다.

**기대 결과 형태**: 각 파일명과 그룹이 줄 단위로 출력된다.

**빈칸 힌트**: csv.DictReader에는 splitlines 결과를 넣는다.

~~~python
manifest = list(csv.DictReader(load_text('____').splitlines()))
for row in manifest[:3]:
    print(row['____'], row['____'])
~~~

---

## 문제 6 — 확장자별 개수 세기

manifest의 type 컬럼을 기준으로 파일 유형별 개수를 센다.

**기대 결과 형태**: md, csv, txt 등 유형별 개수 딕셔너리가 출력된다.

**빈칸 힌트**: Counter 또는 dict 누적을 사용할 수 있다.

~~~python
type_counts = {}
for row in manifest:
    key = row['____']
    type_counts[key] = type_counts.get(key, 0) + 1
print(type_counts)
~~~

---

## 문제 7 — 안전한 파일명 만들기

공백, 슬래시, 한글이 섞인 파일명을 저장 가능한 이름으로 바꾼다.

**기대 결과 형태**: 원본 이름과 정리된 이름이 함께 출력된다.

**빈칸 힌트**: safe_filename 함수에 이름 문자열을 넣는다.

~~~python
names = ['week 1 자료.txt', 'score/template.csv', '학생용 체크리스트.md']
for name in names:
    print(name, '->', ____(name))
~~~

---

## 문제 8 — 하나의 파일 저장하기

lesson_plan.md를 downloads/lesson04/single 폴더에 저장하고 파일 크기를 확인한다.

**기대 결과 형태**: 저장 경로와 0보다 큰 파일 크기가 출력된다.

**빈칸 힌트**: save_bytes는 relative_path와 target_dir을 받는다.

~~~python
target = ____('files/lesson_plan.md', 'downloads/lesson04/single')
print(target)
print(target.____().____)
~~~

---

## 문제 9 — HTML 링크 전체 저장하기

resource_center.html에서 읽은 모든 링크 파일을 한 폴더에 저장한다.

**기대 결과 형태**: 저장된 파일 수와 첫 경로가 출력된다.

**빈칸 힌트**: 각 link의 href를 save_bytes에 넘긴다.

~~~python
saved = []
for link in links:
    saved.append(save_bytes(link['____'], 'downloads/lesson04/all'))
print(len(____))
print(saved[0])
~~~

---

## 문제 10 — 그룹별 폴더로 저장하기

manifest의 group 컬럼을 기준으로 폴더를 나누어 저장한다.

**기대 결과 형태**: 저장 경로 리스트 일부가 출력된다.

**빈칸 힌트**: target_dir은 Path와 group 값을 조합한다.

~~~python
grouped = []
for row in manifest:
    target_dir = Path('downloads/lesson04/by_group') / row['____']
    grouped.append(save_bytes(row['____'], target_dir))
print(grouped[:3])
~~~

---

## 문제 11 — 저장 파일 크기 검증하기

그룹별로 저장한 파일들의 크기를 검사해 빈 파일이 있는지 확인한다.

**기대 결과 형태**: 빈 파일 개수와 전체 파일 수가 출력된다.

**빈칸 힌트**: Path.stat().st_size로 크기를 확인한다.

~~~python
empty_files = []
for target in grouped:
    if target.stat().____ == 0:
        empty_files.append(target)
print('empty:', len(empty_files), 'total:', len(____))
~~~

---

## 문제 12 — 다운로드 로그 만들기

manifest와 저장 경로를 묶어 로그 rows를 만든다.

**기대 결과 형태**: 첫 로그 딕셔너리와 전체 로그 수가 출력된다.

**빈칸 힌트**: zip(manifest, grouped)로 원본 행과 저장 경로를 함께 순회한다.

~~~python
download_log = []
for row, target in zip(____, ____):
    download_log.append({'file_name': row['file_name'], 'group': row['group'], 'saved_path': str(target), 'size': target.stat().st_size})
print(download_log[0])
print(len(download_log))
~~~

---

## 문제 13 — 로그 CSV 저장하기

download_log를 downloads/lesson04/download_log.csv로 저장한다.

**기대 결과 형태**: 저장 파일명과 로그 행 수가 출력된다.

**빈칸 힌트**: DictWriter는 fieldnames와 writeheader가 필요하다.

~~~python
log_path = Path('downloads/lesson04/download_log.csv')
log_path.parent.mkdir(parents=True, exist_ok=True)
with log_path.open('w', newline='', encoding='utf-8') as f:
    writer = csv.DictWriter(f, fieldnames=['file_name', 'group', 'saved_path', 'size'])
    writer.____()
    writer.writerows(____)
print(log_path, len(download_log))
~~~

---

## 문제 14 — 그룹별 저장 개수 검증하기

저장 로그를 다시 읽어 group별 저장 개수를 센다.

**기대 결과 형태**: group별 개수 딕셔너리가 출력된다.

**빈칸 힌트**: 저장한 CSV는 csv.DictReader로 다시 읽는다.

~~~python
loaded_log = list(csv.DictReader(log_path.read_text(encoding='utf-8').splitlines()))
group_counts = Counter(row['____'] for row in loaded_log)
print(dict(____))
~~~

---

## 문제 15 — 다운로드 요약 JSON 저장하기

전체 파일 수, 유형별 개수, 빈 파일 수를 JSON으로 저장한다.

**기대 결과 형태**: 요약 딕셔너리와 저장 파일명이 출력된다.

**빈칸 힌트**: json.dumps와 Path.write_text를 사용한다.

~~~python
summary = {'files': len(____), 'types': dict(type_counts), 'empty': len(empty_files)}
summary_path = Path('downloads/lesson04/download_summary.json')
summary_path.write_text(json.dumps(summary, ensure_ascii=False, indent=2), encoding='utf-8')
print(____)
print(summary_path)
~~~

---

## 문제 풀이 기준

| 구간 | 목표 | 확인할 것 |
|---|---|---|
| 문제 1~3 | HTML 자료실 구조 확인 | h1, a.file-link, href, data 속성 |
| 문제 4~6 | 경로와 manifest 이해 | 원격 URL, 로컬 Path, type 개수 |
| 문제 7~10 | 저장 흐름 완성 | safe_filename, save_bytes, group 폴더 |
| 문제 11~15 | 검증과 산출물 | size, download_log.csv, summary.json |

다운로드 문제는 경로가 조금만 틀려도 실패한다. 오류가 나면 먼저 DATA_BASE를 확인하고, 그다음 href 또는 manifest path가 실제 파일 경로와 맞는지 본다. 저장 문제를 실행한 뒤에는 downloads/lesson04 폴더가 생겼는지, 파일 크기가 0이 아닌지 확인한다.

## 제출 전 자체 점검

- 빈칸을 채운 뒤 위에서 아래로 전체 실행했는가.
- 파일 저장 문제에서 downloads/lesson04 폴더가 생성되는가.
- 저장 파일 수가 manifest 행 수와 일치하는가.
- download_log.csv에 header와 행이 들어 있는가.
- download_summary.json에 전체 파일 수와 empty 값이 들어 있는가.
- 실제 사이트가 아니라 제공된 fixture만 사용했는가.

## 마지막 실행 기준

제출 전에는 런타임을 다시 시작한 뒤 환경 셀부터 마지막 셀까지 한 번에 실행한다. 이전 실행에서 남은 downloads 폴더 때문에 우연히 맞아 보일 수 있으므로, 저장 로그의 행 수와 파일 크기를 반드시 다시 확인한다.

## 오류가 났을 때 확인 순서

1. FileNotFoundError가 나면 manifest의 path가 data/files 안에 실제로 있는지 확인한다.
2. 저장 폴더 오류가 나면 target_dir.mkdir(parents=True, exist_ok=True)가 실행되는지 확인한다.
3. size가 0이면 load_bytes 결과 길이를 먼저 출력한다.
4. CSV 로그가 비어 있으면 download_log 리스트 길이를 출력한다.
5. JSON 저장 오류가 나면 summary 안에 Path 객체처럼 직렬화할 수 없는 값이 들어 있는지 확인한다.

다운로드 문제는 실행 환경과 파일 경로가 얽혀 있어 한 번에 원인을 찾기 어렵다. 경로, 저장, 검증, 로그를 나누어 확인하면 오류 지점을 빠르게 좁힐 수 있다.

## 파일 저장 문제의 채점 포인트

| 항목 | 직접 확인할 값 |
|---|---|
| 저장 개수 | len(saved), len(grouped), len(download_log) |
| 저장 경로 | Path가 downloads/lesson04 아래에 있는지 |
| 파일 크기 | target.stat().st_size가 0보다 큰지 |
| 로그 CSV | header와 행 수가 있는지 |
| 요약 JSON | files, types, empty key가 있는지 |

제출 전에는 downloads/lesson04 폴더를 지우고 다시 실행해도 같은 결과가 나오는지 확인한다. 이전 실행 결과에 기대는 코드는 자동화로 볼 수 없다.

## 다운로드 실습 안전선

이번 문제에서는 제공된 data/files 안의 작은 합성 파일만 저장한다. 외부 사이트 주소를 직접 넣거나 개인 자료가 들어 있는 폴더를 대상으로 실행하지 않는다. 저장 폴더도 downloads/lesson04 아래로 제한한다. 코드를 바꿔 실험하더라도 저장 위치와 파일 개수를 먼저 확인한 뒤 실행한다.

문제 8 이후에는 같은 셀을 여러 번 실행해도 파일이 덮어써질 수 있다. 그래서 파일이 생겼다는 사실만 보지 말고 로그 행 수와 size 값을 다시 확인해야 한다. 자동화에서는 재실행해도 결과가 예측 가능해야 한다.
## 산출물 이름 기준

이번 레슨에서 만드는 파일 이름은 문제와 정확히 맞춘다. 로그는 download_log.csv, 요약은 download_summary.json이다. 이름을 다르게 저장하면 코드가 맞아도 검수자가 결과를 찾기 어렵다.

