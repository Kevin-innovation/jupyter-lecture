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

파일 다운로드와 폴더 정리 레슨의 학생용 문제 노트북이다. 강의 노트북을 먼저 실행한 뒤 빈칸을 직접 채운다.

## 통과 기준

- 총 15문제 중 12문제 이상 정상 출력이면 통과.
- 문제 1~5는 구조 확인, 6~10은 반복 추출, 11~15는 집계와 저장이다.
- 정답값은 적지 않는다. 출력 형태와 HTML/CSV 구조를 보고 직접 판단한다.

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

---

## 문제 1 — 자료실 HTML 열기

자료실 링크 또는 manifest 정보를 사용해 파일 저장 흐름을 완성한다.

**기대 결과 형태**: 경로, 개수, 로그, 요약 문장 중 요구한 결과가 출력된다.

**빈칸 힌트**: HTML 구조, CSV 헤더, 이전 예제를 확인한 뒤 ____ 부분을 직접 채운다.

~~~python
html_text = ____(____)
soup = ____(html_text, 'html.parser')
print(soup.____('____').text.strip())
~~~

---

## 문제 2 — 파일 링크 개수 세기

자료실 링크 또는 manifest 정보를 사용해 파일 저장 흐름을 완성한다.

**기대 결과 형태**: 경로, 개수, 로그, 요약 문장 중 요구한 결과가 출력된다.

**빈칸 힌트**: HTML 구조, CSV 헤더, 이전 예제를 확인한 뒤 ____ 부분을 직접 채운다.

~~~python
links = soup.select('____')
print('links:', ____)
~~~

---

## 문제 3 — 첫 링크 정보 읽기

자료실 링크 또는 manifest 정보를 사용해 파일 저장 흐름을 완성한다.

**기대 결과 형태**: 경로, 개수, 로그, 요약 문장 중 요구한 결과가 출력된다.

**빈칸 힌트**: HTML 구조, CSV 헤더, 이전 예제를 확인한 뒤 ____ 부분을 직접 채운다.

~~~python
first = links[____]
print(first.text.strip())
print(first['____'])
print(first['____'], first['____'])
~~~

---

## 문제 4 — raw URL 만들기

자료실 링크 또는 manifest 정보를 사용해 파일 저장 흐름을 완성한다.

**기대 결과 형태**: 경로, 개수, 로그, 요약 문장 중 요구한 결과가 출력된다.

**빈칸 힌트**: HTML 구조, CSV 헤더, 이전 예제를 확인한 뒤 ____ 부분을 직접 채운다.

~~~python
href = first['href']
if DATA_BASE.startswith('http'):
    print(____(DATA_BASE + '/', href))
else:
    print(Path(DATA_BASE) / href)
~~~

---

## 문제 5 — manifest 읽기

자료실 링크 또는 manifest 정보를 사용해 파일 저장 흐름을 완성한다.

**기대 결과 형태**: 경로, 개수, 로그, 요약 문장 중 요구한 결과가 출력된다.

**빈칸 힌트**: HTML 구조, CSV 헤더, 이전 예제를 확인한 뒤 ____ 부분을 직접 채운다.

~~~python
manifest = list(csv.DictReader(load_text('____').splitlines()))
for row in manifest:
    print(row['____'], row['____'])
~~~

---

## 문제 6 — 확장자별 개수 세기

자료실 링크 또는 manifest 정보를 사용해 파일 저장 흐름을 완성한다.

**기대 결과 형태**: 경로, 개수, 로그, 요약 문장 중 요구한 결과가 출력된다.

**빈칸 힌트**: HTML 구조, CSV 헤더, 이전 예제를 확인한 뒤 ____ 부분을 직접 채운다.

~~~python
type_counts = {}
for row in manifest:
    key = row['____']
    type_counts[key] = type_counts.get(key, 0) + 1
print(type_counts)
~~~

---

## 문제 7 — 안전한 파일명 만들기

자료실 링크 또는 manifest 정보를 사용해 파일 저장 흐름을 완성한다.

**기대 결과 형태**: 경로, 개수, 로그, 요약 문장 중 요구한 결과가 출력된다.

**빈칸 힌트**: HTML 구조, CSV 헤더, 이전 예제를 확인한 뒤 ____ 부분을 직접 채운다.

~~~python
names = ['week 1 자료.txt', 'score/template.csv', '학생용 체크리스트.md']
for name in names:
    print(____(name))
~~~

---

## 문제 8 — 하나의 파일 저장하기

자료실 링크 또는 manifest 정보를 사용해 파일 저장 흐름을 완성한다.

**기대 결과 형태**: 경로, 개수, 로그, 요약 문장 중 요구한 결과가 출력된다.

**빈칸 힌트**: HTML 구조, CSV 헤더, 이전 예제를 확인한 뒤 ____ 부분을 직접 채운다.

~~~python
target = ____('files/lesson_plan.md', 'downloads/lesson04')
print(target)
print(target.____().____)
~~~

---

## 문제 9 — HTML 링크 전체 저장하기

자료실 링크 또는 manifest 정보를 사용해 파일 저장 흐름을 완성한다.

**기대 결과 형태**: 경로, 개수, 로그, 요약 문장 중 요구한 결과가 출력된다.

**빈칸 힌트**: HTML 구조, CSV 헤더, 이전 예제를 확인한 뒤 ____ 부분을 직접 채운다.

~~~python
saved = []
for link in links:
    saved.append(save_bytes(link['____'], 'downloads/lesson04/all'))
print(len(____))
~~~

---

## 문제 10 — 그룹별 폴더로 저장하기

자료실 링크 또는 manifest 정보를 사용해 파일 저장 흐름을 완성한다.

**기대 결과 형태**: 경로, 개수, 로그, 요약 문장 중 요구한 결과가 출력된다.

**빈칸 힌트**: HTML 구조, CSV 헤더, 이전 예제를 확인한 뒤 ____ 부분을 직접 채운다.

~~~python
grouped = []
for row in manifest:
    target_dir = Path('downloads/lesson04/by_group') / row['____']
    grouped.append(save_bytes(row['____'], target_dir))
print(grouped[:3])
~~~

---

## 문제 11 — 저장 파일 크기 검증하기

자료실 링크 또는 manifest 정보를 사용해 파일 저장 흐름을 완성한다.

**기대 결과 형태**: 경로, 개수, 로그, 요약 문장 중 요구한 결과가 출력된다.

**빈칸 힌트**: HTML 구조, CSV 헤더, 이전 예제를 확인한 뒤 ____ 부분을 직접 채운다.

~~~python
empty_files = [p for p in grouped if p.stat().____ == ____]
print(empty_files)
~~~

---

## 문제 12 — CSV 파일만 저장하기

자료실 링크 또는 manifest 정보를 사용해 파일 저장 흐름을 완성한다.

**기대 결과 형태**: 경로, 개수, 로그, 요약 문장 중 요구한 결과가 출력된다.

**빈칸 힌트**: HTML 구조, CSV 헤더, 이전 예제를 확인한 뒤 ____ 부분을 직접 채운다.

~~~python
csv_files = []
for row in manifest:
    if row['____'] == 'csv':
        csv_files.append(save_bytes(row['____'], 'downloads/lesson04/csv'))
print(csv_files)
~~~

---

## 문제 13 — 다운로드 로그 만들기

자료실 링크 또는 manifest 정보를 사용해 파일 저장 흐름을 완성한다.

**기대 결과 형태**: 경로, 개수, 로그, 요약 문장 중 요구한 결과가 출력된다.

**빈칸 힌트**: HTML 구조, CSV 헤더, 이전 예제를 확인한 뒤 ____ 부분을 직접 채운다.

~~~python
logs = []
for row in manifest:
    path = save_bytes(row['path'], Path('downloads/lesson04/logged') / row['group'])
    logs.append({'file_name': row['____'], 'group': row['____'], 'saved_path': str(path), 'size': path.stat().____})
print(logs[0])
print(len(logs))
~~~

---

## 문제 14 — 로그 CSV 저장하기

자료실 링크 또는 manifest 정보를 사용해 파일 저장 흐름을 완성한다.

**기대 결과 형태**: 경로, 개수, 로그, 요약 문장 중 요구한 결과가 출력된다.

**빈칸 힌트**: HTML 구조, CSV 헤더, 이전 예제를 확인한 뒤 ____ 부분을 직접 채운다.

~~~python
with open('lesson04_download_log.csv', 'w', newline='', encoding='utf-8') as f:
    writer = csv.DictWriter(f, fieldnames=['file_name', 'group', 'saved_path', 'size'])
    writer.____()
    writer.____(logs)
print('saved:', 'lesson04_download_log.csv', len(logs))
~~~

---

## 문제 15 — 다운로드 요약 문장 만들기

자료실 링크 또는 manifest 정보를 사용해 파일 저장 흐름을 완성한다.

**기대 결과 형태**: 경로, 개수, 로그, 요약 문장 중 요구한 결과가 출력된다.

**빈칸 힌트**: HTML 구조, CSV 헤더, 이전 예제를 확인한 뒤 ____ 부분을 직접 채운다.

~~~python
total_size = sum(row['____'] for row in logs)
group_count = len(set(row['____'] for row in logs))
print(f'총 {____}개 파일, {total_size} bytes, {group_count}개 그룹으로 정리했습니다.')
~~~

---

### 보강 설명 1

레슨 04 실습 문제은 실행 결과보다 과정 추적이 중요하다. 입력 파일, 반복 단위, selector 또는 경로, 변환 규칙을 분리하면 오류 위치를 빠르게 찾을 수 있다.

### 보강 설명 2

수업 중에는 학생에게 완성 코드를 먼저 보여주지 말고 HTML 구조나 CSV 헤더를 읽게 한다. 구조를 말로 설명할 수 있으면 코드는 짧아진다.

### 보강 설명 3

운영형 자동화는 다음 주에도 다시 실행되어야 한다. 그래서 파일명, URL, 저장 경로, 로그, 요약 문장을 코드 안에서 일관되게 남긴다.

### 보강 설명 4

실제 사이트로 확장할 때는 약관, robots.txt, 요청 간격, 개인정보 여부를 먼저 확인한다. 이 코스의 초반 fixture는 그 안전 습관을 만들기 위한 장치다.

### 보강 설명 5

레슨 04 실습 문제은 실행 결과보다 과정 추적이 중요하다. 입력 파일, 반복 단위, selector 또는 경로, 변환 규칙을 분리하면 오류 위치를 빠르게 찾을 수 있다.

### 보강 설명 6

수업 중에는 학생에게 완성 코드를 먼저 보여주지 말고 HTML 구조나 CSV 헤더를 읽게 한다. 구조를 말로 설명할 수 있으면 코드는 짧아진다.

### 보강 설명 7

운영형 자동화는 다음 주에도 다시 실행되어야 한다. 그래서 파일명, URL, 저장 경로, 로그, 요약 문장을 코드 안에서 일관되게 남긴다.

### 보강 설명 8

실제 사이트로 확장할 때는 약관, robots.txt, 요청 간격, 개인정보 여부를 먼저 확인한다. 이 코스의 초반 fixture는 그 안전 습관을 만들기 위한 장치다.

### 보강 설명 9

레슨 04 실습 문제은 실행 결과보다 과정 추적이 중요하다. 입력 파일, 반복 단위, selector 또는 경로, 변환 규칙을 분리하면 오류 위치를 빠르게 찾을 수 있다.

### 보강 설명 10

수업 중에는 학생에게 완성 코드를 먼저 보여주지 말고 HTML 구조나 CSV 헤더를 읽게 한다. 구조를 말로 설명할 수 있으면 코드는 짧아진다.

### 보강 설명 11

운영형 자동화는 다음 주에도 다시 실행되어야 한다. 그래서 파일명, URL, 저장 경로, 로그, 요약 문장을 코드 안에서 일관되게 남긴다.

### 보강 설명 12

실제 사이트로 확장할 때는 약관, robots.txt, 요청 간격, 개인정보 여부를 먼저 확인한다. 이 코스의 초반 fixture는 그 안전 습관을 만들기 위한 장치다.

### 보강 설명 13

레슨 04 실습 문제은 실행 결과보다 과정 추적이 중요하다. 입력 파일, 반복 단위, selector 또는 경로, 변환 규칙을 분리하면 오류 위치를 빠르게 찾을 수 있다.

### 보강 설명 14

수업 중에는 학생에게 완성 코드를 먼저 보여주지 말고 HTML 구조나 CSV 헤더를 읽게 한다. 구조를 말로 설명할 수 있으면 코드는 짧아진다.

### 보강 설명 15

운영형 자동화는 다음 주에도 다시 실행되어야 한다. 그래서 파일명, URL, 저장 경로, 로그, 요약 문장을 코드 안에서 일관되게 남긴다.

### 보강 설명 16

실제 사이트로 확장할 때는 약관, robots.txt, 요청 간격, 개인정보 여부를 먼저 확인한다. 이 코스의 초반 fixture는 그 안전 습관을 만들기 위한 장치다.

### 보강 설명 17

레슨 04 실습 문제은 실행 결과보다 과정 추적이 중요하다. 입력 파일, 반복 단위, selector 또는 경로, 변환 규칙을 분리하면 오류 위치를 빠르게 찾을 수 있다.

### 보강 설명 18

수업 중에는 학생에게 완성 코드를 먼저 보여주지 말고 HTML 구조나 CSV 헤더를 읽게 한다. 구조를 말로 설명할 수 있으면 코드는 짧아진다.

### 보강 설명 19

운영형 자동화는 다음 주에도 다시 실행되어야 한다. 그래서 파일명, URL, 저장 경로, 로그, 요약 문장을 코드 안에서 일관되게 남긴다.

### 보강 설명 20

실제 사이트로 확장할 때는 약관, robots.txt, 요청 간격, 개인정보 여부를 먼저 확인한다. 이 코스의 초반 fixture는 그 안전 습관을 만들기 위한 장치다.

### 보강 설명 21

레슨 04 실습 문제은 실행 결과보다 과정 추적이 중요하다. 입력 파일, 반복 단위, selector 또는 경로, 변환 규칙을 분리하면 오류 위치를 빠르게 찾을 수 있다.
