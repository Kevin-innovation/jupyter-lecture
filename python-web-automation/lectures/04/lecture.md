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

이 레슨은 자료실 페이지에서 파일 링크를 읽고, manifest 기준으로 폴더를 나누어 저장하는 과정을 다룬다. 자동화에서 다운로드는 단순히 파일을 받는 작업이 아니다. 어떤 파일을 받을지 식별하고, 저장 이름을 안전하게 만들고, 저장 위치와 로그를 남겨 다음 실행 때 누락과 중복을 확인할 수 있어야 한다.

수업 데이터는 모두 합성 fixture다. resource_center.html에는 자료실 링크가 있고, manifest.csv에는 파일명, 그룹, 타입, 주차, 경로가 들어 있다. data/files 폴더의 파일은 실제 업무 자료를 흉내 낸 작은 샘플 파일이다.

## 학습 목표

1. HTML의 a.file-link에서 href와 data 속성을 읽는다.
2. bytes 데이터를 파일로 저장하고 파일 크기를 확인한다.
3. 파일명을 안전한 저장 이름으로 정리한다.
4. manifest의 group, type, week 기준으로 폴더를 나눈다.
5. 저장 성공 여부와 파일 크기를 CSV 로그로 남긴다.

---

## 1. 환경 준비

코랩에서는 GitHub raw URL에서 fixture를 읽고, 로컬에서는 현재 레슨 폴더의 data 디렉터리를 읽는다. 다운로드 자동화는 파일 경로 오류가 자주 나므로 DATA_BASE 출력부터 확인한다.

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

다운로드 실습에서는 load_text와 load_bytes를 구분한다. HTML, CSV, JSON처럼 텍스트로 읽을 수 있는 자료는 load_text를 쓰고, 실제 파일 저장에는 load_bytes를 쓴다. 모든 파일을 텍스트로 읽으면 인코딩이 맞지 않는 파일에서 오류가 날 수 있다.

---

## 2. 자료실 HTML에서 링크 읽기

resource_center.html은 수업 자료실 화면을 흉내 낸 fixture다. 각 파일 링크는 a.file-link 태그이고, href에는 파일 경로, data-type에는 확장자 유형, data-week에는 주차, data-group에는 업무 그룹이 들어 있다.

~~~python
resource_soup = read_soup('resource_center.html')
heading = resource_soup.select_one('h1').text.strip()
links = resource_soup.select('a.file-link')
print(heading)
print('link count:', len(links))
print(links[0].text.strip(), links[0]['href'], links[0]['data-type'], links[0]['data-group'])
~~~

파일 링크를 읽을 때는 화면 텍스트와 href를 분리해서 봐야 한다. 텍스트는 사람이 보는 파일명이고, href는 실제로 읽어야 할 경로다. 둘이 항상 같다고 가정하면 실제 사이트에서 쉽게 깨진다.

---

## 3. 링크 정보를 레코드로 정리하기

HTML 링크를 바로 저장하지 말고 먼저 딕셔너리 목록으로 정리한다. 이 단계에서 파일명, 경로, 타입, 주차, 그룹을 명확히 분리하면 이후 필터링과 폴더 분류가 쉬워진다.

~~~python
link_records = []
for link in links:
    link_records.append({
        'file_name': link.text.strip(),
        'href': link['href'],
        'type': link['data-type'],
        'week': int(link['data-week']),
        'group': link['data-group'],
    })
print(link_records[:3])
~~~

week는 계산과 필터링에 사용할 수 있으므로 int로 바꾼다. type과 group은 폴더 분류 기준으로 사용할 수 있다. 원본 href는 저장 함수에 넘길 실제 입력이므로 그대로 보존한다.

---

## 4. 원격 경로와 로컬 경로 구분하기

코랩에서는 DATA_BASE가 GitHub raw URL이고, 로컬에서는 ./data다. 같은 href라도 실행 환경에 따라 실제 읽는 위치가 달라진다. 다운로드 URL을 표시할 때는 urljoin을 사용해 상대 경로를 안전하게 합친다.

~~~python
first_href = link_records[0]['href']
if DATA_BASE.startswith('http'):
    source_location = urljoin(DATA_BASE + '/', first_href)
else:
    source_location = Path(DATA_BASE) / first_href
print(source_location)
~~~

상대 경로를 문자열 덧셈으로 붙이면 슬래시가 빠지거나 중복되기 쉽다. urljoin과 Path를 실행 환경에 맞게 사용하면 같은 코드가 코랩과 로컬에서 모두 안정적으로 동작한다.

---

## 5. 파일명을 안전하게 정리하기

웹에서 받은 파일명에는 공백, 괄호, 슬래시, 한글, 특수문자가 섞일 수 있다. 저장 이름은 운영체제와 자동화 로그에서 다루기 쉬워야 한다. safe_filename은 경로 성분을 제거하고 허용 문자만 남긴다.

~~~python
sample_names = ['week 1 자료.txt', 'score/template.csv', '학생용 체크리스트.md', 'report(final).html']
for name in sample_names:
    print(name, '->', safe_filename(name))
~~~

실제 운영에서는 원본 이름과 저장 이름을 둘 다 로그에 남긴다. 원본 이름은 사람이 이해하는 데 필요하고, 저장 이름은 재실행과 파일 시스템 관리에 필요하다.

---

## 6. bytes로 하나의 파일 저장하기

다운로드 대상이 텍스트 파일처럼 보여도 저장은 bytes 기준으로 처리하는 편이 안전하다. save_bytes는 파일을 bytes로 읽고 target_dir을 만든 뒤 안전한 이름으로 저장한다.

~~~python
single_target = save_bytes('files/lesson_plan.md', 'downloads/lesson04/single')
print(file_info(single_target))
print(single_target.read_text(encoding='utf-8')[:40])
~~~

파일 저장 후에는 존재 여부와 크기를 확인한다. 크기가 0이면 다운로드는 성공처럼 보여도 실제 파일은 비어 있을 수 있다.

---

## 7. manifest.csv 읽기

manifest.csv는 다운로드할 파일 목록을 관리하는 표다. HTML 링크를 기준으로 받아도 되지만, 운영 자동화에서는 manifest처럼 관리용 목록을 기준으로 삼는 편이 더 명확할 때가 많다.

~~~python
manifest = list(csv.DictReader(load_text('manifest.csv').splitlines()))
print(manifest[0])
print('manifest rows:', len(manifest))
print('groups:', sorted(set(row['group'] for row in manifest)))
~~~

manifest에는 file_name, group, type, week, path가 들어 있다. path는 실제 파일 경로이고, group은 폴더 분류 기준이다. file_name은 로그에 남길 사람이 읽는 이름이다.

---

## 8. 타입과 그룹별 개수 세기

다운로드 전에 어떤 파일이 몇 개인지 요약하면 저장 결과를 검증하기 쉽다. manifest 기준으로 type과 group 개수를 세어 보자.

~~~python
type_counts = Counter(row['type'] for row in manifest)
group_counts = Counter(row['group'] for row in manifest)
week_counts = Counter(row['week'] for row in manifest)
print('type:', dict(type_counts))
print('group:', dict(group_counts))
print('week:', dict(week_counts))
~~~

저장 후 파일 개수와 이 요약값이 맞아야 한다. 예를 들어 group별 폴더에 저장했다면 score 폴더에는 manifest에서 score로 표시된 파일 수만큼 들어 있어야 한다.

---

## 9. 그룹별 폴더로 저장하기

파일을 한 폴더에 모두 넣으면 나중에 찾기 어렵다. 이번 레슨에서는 downloads/lesson04/by_group 아래에 group별 폴더를 만들고 파일을 저장한다.

~~~python
grouped_targets = []
for row in manifest:
    target_dir = Path('downloads/lesson04/by_group') / row['group']
    target = save_bytes(row['path'], target_dir)
    grouped_targets.append(target)
print('saved:', len(grouped_targets))
print(grouped_targets[:4])
~~~

폴더 기준은 업무에 따라 달라질 수 있다. 수업 운영에서는 group, week, type이 자주 쓰인다. 중요한 것은 기준을 코드와 로그에 같이 남기는 것이다.

---

## 10. 저장 로그 만들기

다운로드 자동화는 성공한 파일 목록을 로그로 남겨야 한다. 여기서는 원본 파일명, 그룹, 타입, 주차, 저장 경로, 파일 크기를 CSV로 저장한다.

~~~python
download_log = []
for row, target in zip(manifest, grouped_targets):
    info = file_info(target)
    download_log.append({
        'file_name': row['file_name'],
        'group': row['group'],
        'type': row['type'],
        'week': row['week'],
        'saved_path': info['path'],
        'size': info['size'],
        'status': 'saved' if info['size'] > 0 else 'empty',
    })
print(download_log[:3])
~~~

로그는 다음 실행에서 중복 저장을 피하거나 실패 파일만 다시 처리하는 기준이 된다. 단순히 print만 남기면 나중에 자동화 결과를 확인할 방법이 없다.

---

## 11. 로그 CSV 저장과 검증

저장 로그도 산출물이다. CSV로 저장한 뒤 다시 읽어서 행 수와 상태 분포를 확인한다. 저장한 파일을 다시 읽는 과정이 있어야 결과가 실제로 남았는지 확인할 수 있다.

~~~python
log_path = Path('downloads/lesson04/download_log.csv')
log_path.parent.mkdir(parents=True, exist_ok=True)
with log_path.open('w', newline='', encoding='utf-8') as f:
    writer = csv.DictWriter(f, fieldnames=['file_name', 'group', 'type', 'week', 'saved_path', 'size', 'status'])
    writer.writeheader()
    writer.writerows(download_log)

loaded_log = list(csv.DictReader(log_path.read_text(encoding='utf-8').splitlines()))
print('log rows:', len(loaded_log))
print('status:', Counter(row['status'] for row in loaded_log))
~~~

검증은 저장 직후에 한다. 파일이 존재하는지, 행 수가 manifest와 같은지, size가 0보다 큰지 확인하면 기본적인 다운로드 누락을 잡을 수 있다.

---

## 12. 중복 저장 방지 기준

같은 파일을 반복 저장하면 폴더가 지저분해지고 로그도 믿기 어려워진다. 이번 fixture에서는 file_name 또는 path를 중복 기준으로 사용할 수 있다. 실제 서비스에서는 URL, 파일 크기, 수정일을 함께 쓰는 경우가 많다.

~~~python
seen_paths = set()
duplicates = []
for row in manifest:
    if row['path'] in seen_paths:
        duplicates.append(row['path'])
    seen_paths.add(row['path'])
print('duplicates:', duplicates)
print('unique paths:', len(seen_paths))
~~~

중복이 없다는 확인도 로그의 일부가 될 수 있다. 자동화는 성공한 작업뿐 아니라 검증한 기준도 남겨야 운영자가 믿고 사용할 수 있다.

---

## 13. 실제 사이트 확장 전 안전 기준

자료실 다운로드는 실제 서비스에서 특히 조심해야 한다. 대용량 파일을 반복 다운로드하거나 비공개 자료를 저장하면 시스템과 개인정보 측면에서 문제가 생길 수 있다.

수업에서는 다음 기준을 적용한다.

- 공개된 수업용 자료나 허가된 내부 자료만 다룬다.
- 파일 개수와 최대 용량을 제한한다.
- 같은 URL을 짧은 시간에 반복 요청하지 않는다.
- 저장 위치와 접근 권한을 정한다.
- 실패 파일과 건너뛴 파일을 로그에 남긴다.

~~~python
safety_rules = {
    'max_files': 20,
    'allowed_source': 'lesson fixture only',
    'log_required': True,
    'private_data': 'not allowed in practice fixture',
}
print(safety_rules)
~~~

수업용 자동화에서도 이 기준을 말로 설명하게 해야 한다. 그래야 학생이 실제 사이트에서 무분별하게 다운로드 코드를 실행하지 않는다.

---

## 수업 정리

4강의 핵심은 파일을 받는 코드보다 저장 전후의 기준이다. 링크를 읽고, 경로를 만들고, bytes로 저장하고, 폴더를 나누고, 로그를 남기는 순서가 분명해야 한다. 다운로드 자동화는 다음 주에도 다시 실행될 가능성이 높으므로 재실행 기준과 검증 기준을 함께 설계해야 한다.

---

## 14. 다운로드 검수 루틴

다운로드 자동화에서 가장 흔한 착각은 파일이 생겼으니 성공했다고 보는 것이다. 실제 검수는 더 구체적이어야 한다. 저장된 파일이 존재하는지, 크기가 0보다 큰지, manifest 행 수와 저장 행 수가 같은지, 저장 폴더 기준이 의도와 맞는지를 확인한다.

~~~python
expected_count = len(manifest)
actual_count = len(grouped_targets)
empty_count = sum(1 for target in grouped_targets if target.stat().st_size == 0)
print('expected:', expected_count)
print('actual:', actual_count)
print('empty:', empty_count)
print('ok:', expected_count == actual_count and empty_count == 0)
~~~

이 검수 루틴은 파일 개수가 많은 실제 업무에서 특히 중요하다. 다운로드 중 일부만 실패해도 전체 자동화는 성공처럼 보일 수 있다. 그러므로 성공 조건을 코드로 분명히 적어야 한다. 예를 들어 저장 개수가 manifest 개수와 같고, 빈 파일이 없고, 로그 CSV 행 수가 저장 개수와 같아야 성공으로 볼 수 있다.

---

## 15. 폴더 기준을 바꿔 저장하기

앞에서는 group 기준으로 저장했다. 수업 운영에서는 week 기준이나 type 기준이 더 편할 때도 있다. 기준이 바뀌어도 저장 함수는 그대로 두고 target_dir만 바꾸면 된다.

~~~python
week_targets = []
for row in manifest:
    target_dir = Path('downloads/lesson04/by_week') / f"week_{row['week']}"
    week_targets.append(save_bytes(row['path'], target_dir))
print('week folders:', sorted({str(target.parent) for target in week_targets}))

type_targets = []
for row in manifest:
    target_dir = Path('downloads/lesson04/by_type') / row['type']
    type_targets.append(save_bytes(row['path'], target_dir))
print('type folders:', sorted({str(target.parent) for target in type_targets}))
~~~

학생에게 같은 데이터를 여러 기준으로 저장해 보게 하면 폴더 설계가 자동화 결과의 사용성에 영향을 준다는 점을 이해한다. group 기준은 업무 목적별로 찾기 쉽고, week 기준은 수업 회차별로 정리하기 좋고, type 기준은 파일 형식별 후처리에 적합하다.

---

## 16. 실패를 로그로 남기는 방식

수업 fixture는 모두 존재하지만 실제 사이트에서는 일부 파일이 삭제되거나 권한이 없거나 네트워크가 끊길 수 있다. 자동화는 실패를 숨기지 말고 실패한 항목을 별도 상태로 남겨야 한다.

~~~python
def try_save(row, target_root):
    try:
        target = save_bytes(row['path'], Path(target_root) / row['group'])
        return {
            'file_name': row['file_name'],
            'source_path': row['path'],
            'saved_path': str(target),
            'size': target.stat().st_size,
            'status': 'saved' if target.stat().st_size > 0 else 'empty',
            'error': '',
        }
    except Exception as exc:
        return {
            'file_name': row.get('file_name', ''),
            'source_path': row.get('path', ''),
            'saved_path': '',
            'size': 0,
            'status': 'failed',
            'error': type(exc).__name__,
        }

checked_log = [try_save(row, 'downloads/lesson04/checked') for row in manifest]
print(Counter(row['status'] for row in checked_log))
~~~

예외를 무조건 무시하면 나중에 어떤 파일이 누락되었는지 알 수 없다. 반대로 예외가 하나 났다고 전체 자동화를 중단하면 나머지 파일까지 처리하지 못한다. 운영형 다운로드에서는 실패를 기록하고 마지막에 실패 목록을 보고하는 방식이 더 적합하다.

---

## 17. 실제 운영으로 확장할 때 바뀌는 점

fixture에서는 파일이 작고 권한 문제가 없다. 실제 자료실에서는 로그인, 세션, 파일 크기, 중복 파일명, 저장 권한이 함께 등장한다. 수업에서는 코드를 복잡하게 만들지 않지만, 운영 설계 질문은 미리 던져야 한다.

| 질문 | 이유 | 수업에서의 대응 |
|---|---|---|
| 파일을 받을 권한이 있는가 | 비공개 자료 유출 방지 | fixture만 사용 |
| 파일 개수와 용량 제한이 있는가 | 서버와 로컬 저장 공간 보호 | max_files 기준 작성 |
| 같은 파일을 또 받는가 | 중복 저장 방지 | manifest path를 key로 사용 |
| 실패 파일은 어디에 남기는가 | 재시도와 누락 점검 | status, error 로그 작성 |
| 저장 폴더를 누가 보는가 | 접근 권한 관리 | downloads 하위 폴더로 제한 |

다운로드 자동화는 편리하지만 잘못 만들면 자료 유출이나 서버 부하로 이어질 수 있다. 그래서 4강의 목표는 빠른 다운로드가 아니라 통제 가능한 다운로드 흐름을 만드는 것이다.

