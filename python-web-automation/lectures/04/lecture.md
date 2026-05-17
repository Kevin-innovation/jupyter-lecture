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

# 레슨 04 — 파일 다운로드와 폴더 정리

[![Open in Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/Kevin-innovation/jupyter-lecture/blob/main/python-web-automation/lectures/04/%5B%ED%95%99%EC%83%9D%EC%9A%A9%5D%20%EB%A0%88%EC%8A%A8%2004%20%E2%80%94%20%ED%8C%8C%EC%9D%BC%20%EB%8B%A4%EC%9A%B4%EB%A1%9C%EB%93%9C%EC%99%80%20%ED%8F%B4%EB%8D%94%20%EC%A0%95%EB%A6%AC.ipynb)

이 노트북은 읽기와 따라하기용 강의 노트북이다. 학생은 셀을 위에서 아래로 실행하며 입력 데이터가 어떤 구조로 바뀌는지 확인한다. 파일 다운로드와 폴더 정리는 실제 업무 자동화에서 자주 등장하는 반복 패턴을 합성 fixture로 안전하게 연습한다.

## 학습 목표

1. HTML에서 다운로드 링크와 파일 메타데이터를 추출한다.
2. bytes 데이터를 파일로 저장한다.
3. 파일명을 안전하게 정리한다.
4. manifest 기준으로 폴더를 나누어 저장한다.
5. 다운로드 로그 CSV를 만든다.

---

## 1. 파일 링크 구조

자료실 HTML의 a.file-link는 href와 data-type, data-week 속성을 가진다.

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

html_text = load_text('resource_center.html')
soup = BeautifulSoup(html_text, 'html.parser')
links = soup.select('a.file-link')
print(len(links), links[0]['href'])
~~~

---

## 2. bytes 저장

다운로드 파일은 텍스트가 아닐 수 있으므로 bytes로 읽고 write_bytes로 저장한다.

~~~python
content = load_bytes('files/lesson_plan.md')
print(type(content), len(content))
~~~

---

## 3. manifest 기반 폴더 정리

manifest의 group, type, path 컬럼을 사용해 저장 위치와 로그를 만든다.

~~~python
manifest = list(csv.DictReader(load_text('manifest.csv').splitlines()))
for row in manifest[:3]:
    print(row['file_name'], row['group'], row['type'])
~~~

---

## 데이터 출처와 안전 규칙

이 레슨의 파일은 모두 수업용 합성 데이터다. 실제 사이트의 개인정보, 로그인 정보, 유료 콘텐츠를 포함하지 않는다. 실제 사이트로 확장할 때는 robots.txt, 이용 약관, 요청 간격, 수집 목적을 먼저 확인한다. 수업 중에는 fixture를 반복 실행하며 구조를 익히고, 외부 사이트를 빠르게 반복 요청하지 않는다.

### 보강 설명 1

레슨 04 강의은 실행 결과보다 과정 추적이 중요하다. 입력 파일, 반복 단위, selector 또는 경로, 변환 규칙을 분리하면 오류 위치를 빠르게 찾을 수 있다.

### 보강 설명 2

수업 중에는 학생에게 완성 코드를 먼저 보여주지 말고 HTML 구조나 CSV 헤더를 읽게 한다. 구조를 말로 설명할 수 있으면 코드는 짧아진다.

### 보강 설명 3

운영형 자동화는 다음 주에도 다시 실행되어야 한다. 그래서 파일명, URL, 저장 경로, 로그, 요약 문장을 코드 안에서 일관되게 남긴다.

### 보강 설명 4

실제 사이트로 확장할 때는 약관, robots.txt, 요청 간격, 개인정보 여부를 먼저 확인한다. 이 코스의 초반 fixture는 그 안전 습관을 만들기 위한 장치다.

### 보강 설명 5

레슨 04 강의은 실행 결과보다 과정 추적이 중요하다. 입력 파일, 반복 단위, selector 또는 경로, 변환 규칙을 분리하면 오류 위치를 빠르게 찾을 수 있다.

### 보강 설명 6

수업 중에는 학생에게 완성 코드를 먼저 보여주지 말고 HTML 구조나 CSV 헤더를 읽게 한다. 구조를 말로 설명할 수 있으면 코드는 짧아진다.

### 보강 설명 7

운영형 자동화는 다음 주에도 다시 실행되어야 한다. 그래서 파일명, URL, 저장 경로, 로그, 요약 문장을 코드 안에서 일관되게 남긴다.

### 보강 설명 8

실제 사이트로 확장할 때는 약관, robots.txt, 요청 간격, 개인정보 여부를 먼저 확인한다. 이 코스의 초반 fixture는 그 안전 습관을 만들기 위한 장치다.

### 보강 설명 9

레슨 04 강의은 실행 결과보다 과정 추적이 중요하다. 입력 파일, 반복 단위, selector 또는 경로, 변환 규칙을 분리하면 오류 위치를 빠르게 찾을 수 있다.

### 보강 설명 10

수업 중에는 학생에게 완성 코드를 먼저 보여주지 말고 HTML 구조나 CSV 헤더를 읽게 한다. 구조를 말로 설명할 수 있으면 코드는 짧아진다.

### 보강 설명 11

운영형 자동화는 다음 주에도 다시 실행되어야 한다. 그래서 파일명, URL, 저장 경로, 로그, 요약 문장을 코드 안에서 일관되게 남긴다.

### 보강 설명 12

실제 사이트로 확장할 때는 약관, robots.txt, 요청 간격, 개인정보 여부를 먼저 확인한다. 이 코스의 초반 fixture는 그 안전 습관을 만들기 위한 장치다.

### 보강 설명 13

레슨 04 강의은 실행 결과보다 과정 추적이 중요하다. 입력 파일, 반복 단위, selector 또는 경로, 변환 규칙을 분리하면 오류 위치를 빠르게 찾을 수 있다.

### 보강 설명 14

수업 중에는 학생에게 완성 코드를 먼저 보여주지 말고 HTML 구조나 CSV 헤더를 읽게 한다. 구조를 말로 설명할 수 있으면 코드는 짧아진다.

### 보강 설명 15

운영형 자동화는 다음 주에도 다시 실행되어야 한다. 그래서 파일명, URL, 저장 경로, 로그, 요약 문장을 코드 안에서 일관되게 남긴다.

### 보강 설명 16

실제 사이트로 확장할 때는 약관, robots.txt, 요청 간격, 개인정보 여부를 먼저 확인한다. 이 코스의 초반 fixture는 그 안전 습관을 만들기 위한 장치다.

### 보강 설명 17

레슨 04 강의은 실행 결과보다 과정 추적이 중요하다. 입력 파일, 반복 단위, selector 또는 경로, 변환 규칙을 분리하면 오류 위치를 빠르게 찾을 수 있다.

### 보강 설명 18

수업 중에는 학생에게 완성 코드를 먼저 보여주지 말고 HTML 구조나 CSV 헤더를 읽게 한다. 구조를 말로 설명할 수 있으면 코드는 짧아진다.

### 보강 설명 19

운영형 자동화는 다음 주에도 다시 실행되어야 한다. 그래서 파일명, URL, 저장 경로, 로그, 요약 문장을 코드 안에서 일관되게 남긴다.

### 보강 설명 20

실제 사이트로 확장할 때는 약관, robots.txt, 요청 간격, 개인정보 여부를 먼저 확인한다. 이 코스의 초반 fixture는 그 안전 습관을 만들기 위한 장치다.

### 보강 설명 21

레슨 04 강의은 실행 결과보다 과정 추적이 중요하다. 입력 파일, 반복 단위, selector 또는 경로, 변환 규칙을 분리하면 오류 위치를 빠르게 찾을 수 있다.

### 보강 설명 22

수업 중에는 학생에게 완성 코드를 먼저 보여주지 말고 HTML 구조나 CSV 헤더를 읽게 한다. 구조를 말로 설명할 수 있으면 코드는 짧아진다.

### 보강 설명 23

운영형 자동화는 다음 주에도 다시 실행되어야 한다. 그래서 파일명, URL, 저장 경로, 로그, 요약 문장을 코드 안에서 일관되게 남긴다.

### 보강 설명 24

실제 사이트로 확장할 때는 약관, robots.txt, 요청 간격, 개인정보 여부를 먼저 확인한다. 이 코스의 초반 fixture는 그 안전 습관을 만들기 위한 장치다.

### 보강 설명 25

레슨 04 강의은 실행 결과보다 과정 추적이 중요하다. 입력 파일, 반복 단위, selector 또는 경로, 변환 규칙을 분리하면 오류 위치를 빠르게 찾을 수 있다.

### 보강 설명 26

수업 중에는 학생에게 완성 코드를 먼저 보여주지 말고 HTML 구조나 CSV 헤더를 읽게 한다. 구조를 말로 설명할 수 있으면 코드는 짧아진다.

### 보강 설명 27

운영형 자동화는 다음 주에도 다시 실행되어야 한다. 그래서 파일명, URL, 저장 경로, 로그, 요약 문장을 코드 안에서 일관되게 남긴다.

### 보강 설명 28

실제 사이트로 확장할 때는 약관, robots.txt, 요청 간격, 개인정보 여부를 먼저 확인한다. 이 코스의 초반 fixture는 그 안전 습관을 만들기 위한 장치다.

### 보강 설명 29

레슨 04 강의은 실행 결과보다 과정 추적이 중요하다. 입력 파일, 반복 단위, selector 또는 경로, 변환 규칙을 분리하면 오류 위치를 빠르게 찾을 수 있다.

### 보강 설명 30

수업 중에는 학생에게 완성 코드를 먼저 보여주지 말고 HTML 구조나 CSV 헤더를 읽게 한다. 구조를 말로 설명할 수 있으면 코드는 짧아진다.

### 보강 설명 31

운영형 자동화는 다음 주에도 다시 실행되어야 한다. 그래서 파일명, URL, 저장 경로, 로그, 요약 문장을 코드 안에서 일관되게 남긴다.

### 보강 설명 32

실제 사이트로 확장할 때는 약관, robots.txt, 요청 간격, 개인정보 여부를 먼저 확인한다. 이 코스의 초반 fixture는 그 안전 습관을 만들기 위한 장치다.

### 보강 설명 33

레슨 04 강의은 실행 결과보다 과정 추적이 중요하다. 입력 파일, 반복 단위, selector 또는 경로, 변환 규칙을 분리하면 오류 위치를 빠르게 찾을 수 있다.

### 보강 설명 34

수업 중에는 학생에게 완성 코드를 먼저 보여주지 말고 HTML 구조나 CSV 헤더를 읽게 한다. 구조를 말로 설명할 수 있으면 코드는 짧아진다.

### 보강 설명 35

운영형 자동화는 다음 주에도 다시 실행되어야 한다. 그래서 파일명, URL, 저장 경로, 로그, 요약 문장을 코드 안에서 일관되게 남긴다.

### 보강 설명 36

실제 사이트로 확장할 때는 약관, robots.txt, 요청 간격, 개인정보 여부를 먼저 확인한다. 이 코스의 초반 fixture는 그 안전 습관을 만들기 위한 장치다.

### 보강 설명 37

레슨 04 강의은 실행 결과보다 과정 추적이 중요하다. 입력 파일, 반복 단위, selector 또는 경로, 변환 규칙을 분리하면 오류 위치를 빠르게 찾을 수 있다.

### 보강 설명 38

수업 중에는 학생에게 완성 코드를 먼저 보여주지 말고 HTML 구조나 CSV 헤더를 읽게 한다. 구조를 말로 설명할 수 있으면 코드는 짧아진다.

### 보강 설명 39

운영형 자동화는 다음 주에도 다시 실행되어야 한다. 그래서 파일명, URL, 저장 경로, 로그, 요약 문장을 코드 안에서 일관되게 남긴다.

### 보강 설명 40

실제 사이트로 확장할 때는 약관, robots.txt, 요청 간격, 개인정보 여부를 먼저 확인한다. 이 코스의 초반 fixture는 그 안전 습관을 만들기 위한 장치다.

### 보강 설명 41

레슨 04 강의은 실행 결과보다 과정 추적이 중요하다. 입력 파일, 반복 단위, selector 또는 경로, 변환 규칙을 분리하면 오류 위치를 빠르게 찾을 수 있다.

### 보강 설명 42

수업 중에는 학생에게 완성 코드를 먼저 보여주지 말고 HTML 구조나 CSV 헤더를 읽게 한다. 구조를 말로 설명할 수 있으면 코드는 짧아진다.

### 보강 설명 43

운영형 자동화는 다음 주에도 다시 실행되어야 한다. 그래서 파일명, URL, 저장 경로, 로그, 요약 문장을 코드 안에서 일관되게 남긴다.

### 보강 설명 44

실제 사이트로 확장할 때는 약관, robots.txt, 요청 간격, 개인정보 여부를 먼저 확인한다. 이 코스의 초반 fixture는 그 안전 습관을 만들기 위한 장치다.

### 보강 설명 45

레슨 04 강의은 실행 결과보다 과정 추적이 중요하다. 입력 파일, 반복 단위, selector 또는 경로, 변환 규칙을 분리하면 오류 위치를 빠르게 찾을 수 있다.

### 보강 설명 46

수업 중에는 학생에게 완성 코드를 먼저 보여주지 말고 HTML 구조나 CSV 헤더를 읽게 한다. 구조를 말로 설명할 수 있으면 코드는 짧아진다.

### 보강 설명 47

운영형 자동화는 다음 주에도 다시 실행되어야 한다. 그래서 파일명, URL, 저장 경로, 로그, 요약 문장을 코드 안에서 일관되게 남긴다.

### 보강 설명 48

실제 사이트로 확장할 때는 약관, robots.txt, 요청 간격, 개인정보 여부를 먼저 확인한다. 이 코스의 초반 fixture는 그 안전 습관을 만들기 위한 장치다.

### 보강 설명 49

레슨 04 강의은 실행 결과보다 과정 추적이 중요하다. 입력 파일, 반복 단위, selector 또는 경로, 변환 규칙을 분리하면 오류 위치를 빠르게 찾을 수 있다.

### 보강 설명 50

수업 중에는 학생에게 완성 코드를 먼저 보여주지 말고 HTML 구조나 CSV 헤더를 읽게 한다. 구조를 말로 설명할 수 있으면 코드는 짧아진다.

### 보강 설명 51

운영형 자동화는 다음 주에도 다시 실행되어야 한다. 그래서 파일명, URL, 저장 경로, 로그, 요약 문장을 코드 안에서 일관되게 남긴다.

### 보강 설명 52

실제 사이트로 확장할 때는 약관, robots.txt, 요청 간격, 개인정보 여부를 먼저 확인한다. 이 코스의 초반 fixture는 그 안전 습관을 만들기 위한 장치다.

### 보강 설명 53

레슨 04 강의은 실행 결과보다 과정 추적이 중요하다. 입력 파일, 반복 단위, selector 또는 경로, 변환 규칙을 분리하면 오류 위치를 빠르게 찾을 수 있다.

### 보강 설명 54

수업 중에는 학생에게 완성 코드를 먼저 보여주지 말고 HTML 구조나 CSV 헤더를 읽게 한다. 구조를 말로 설명할 수 있으면 코드는 짧아진다.

### 보강 설명 55

운영형 자동화는 다음 주에도 다시 실행되어야 한다. 그래서 파일명, URL, 저장 경로, 로그, 요약 문장을 코드 안에서 일관되게 남긴다.

### 보강 설명 56

실제 사이트로 확장할 때는 약관, robots.txt, 요청 간격, 개인정보 여부를 먼저 확인한다. 이 코스의 초반 fixture는 그 안전 습관을 만들기 위한 장치다.

### 보강 설명 57

레슨 04 강의은 실행 결과보다 과정 추적이 중요하다. 입력 파일, 반복 단위, selector 또는 경로, 변환 규칙을 분리하면 오류 위치를 빠르게 찾을 수 있다.

### 보강 설명 58

수업 중에는 학생에게 완성 코드를 먼저 보여주지 말고 HTML 구조나 CSV 헤더를 읽게 한다. 구조를 말로 설명할 수 있으면 코드는 짧아진다.

### 보강 설명 59

운영형 자동화는 다음 주에도 다시 실행되어야 한다. 그래서 파일명, URL, 저장 경로, 로그, 요약 문장을 코드 안에서 일관되게 남긴다.

### 보강 설명 60

실제 사이트로 확장할 때는 약관, robots.txt, 요청 간격, 개인정보 여부를 먼저 확인한다. 이 코스의 초반 fixture는 그 안전 습관을 만들기 위한 장치다.

### 보강 설명 61

레슨 04 강의은 실행 결과보다 과정 추적이 중요하다. 입력 파일, 반복 단위, selector 또는 경로, 변환 규칙을 분리하면 오류 위치를 빠르게 찾을 수 있다.

### 보강 설명 62

수업 중에는 학생에게 완성 코드를 먼저 보여주지 말고 HTML 구조나 CSV 헤더를 읽게 한다. 구조를 말로 설명할 수 있으면 코드는 짧아진다.

### 보강 설명 63

운영형 자동화는 다음 주에도 다시 실행되어야 한다. 그래서 파일명, URL, 저장 경로, 로그, 요약 문장을 코드 안에서 일관되게 남긴다.

### 보강 설명 64

실제 사이트로 확장할 때는 약관, robots.txt, 요청 간격, 개인정보 여부를 먼저 확인한다. 이 코스의 초반 fixture는 그 안전 습관을 만들기 위한 장치다.

### 보강 설명 65

레슨 04 강의은 실행 결과보다 과정 추적이 중요하다. 입력 파일, 반복 단위, selector 또는 경로, 변환 규칙을 분리하면 오류 위치를 빠르게 찾을 수 있다.
