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

# 레슨 10 — 최종 미션 모범 답안

[![Open in Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/Kevin-innovation/jupyter-lecture/blob/main/python-web-automation/lectures/10/%EB%A0%88%EC%8A%A8%2010%20%E2%80%94%20%ED%86%B5%ED%95%A9%20%ED%94%84%EB%A1%9C%EC%A0%9D%ED%8A%B8%3A%20%EC%9E%90%EB%8F%99%ED%99%94%20%EB%A6%AC%ED%8F%AC%ED%8A%B8%20%EB%A7%8C%EB%93%A4%EA%B8%B0.ipynb)

> 선생님용 최종 미션 모범 답안이다. 코랩에서 확인하려면 우측 상단 또는 아래의 **Open in Colab** 버튼을 클릭한다.


> 교사 확인용 모범 답안이다. 학생에게는 최종 미션 조건만 제공한다.

## 실행 코드

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


index = parse_html('portal_index.html')
print(text_or_empty(index.select_one('h1')))

links = []
for a in index.select('a[data-page]'):
    links.append({'label': a.get_text(' ', strip=True), 'href': a['href']})
print(links)

base = 'https://lesson.local/portal/'
absolute = [urljoin(base, item['href']) for item in links]
print(absolute)

notice = parse_html('portal_notice.html')
notice_cards = notice.select('.notice-card')
print(len(notice_cards))

notices = []
for card in notice_cards:
    notices.append({'title': text_or_empty(card.select_one('.title')), 'level': card.get('data-level'), 'date': card.get('data-date')})
print(notices[:3])

courses = parse_html('portal_courses.html')
course_rows = []
for row in courses.select('tbody tr'):
    cells = [td.get_text(' ', strip=True) for td in row.select('td')]
    course_rows.append({'course': cells[0], 'teacher': cells[1], 'students': clean_int(cells[2]), 'status': cells[3]})
print(course_rows[:2])

manifest = load_csv('download_manifest.csv')
print(manifest[0]['filename'])

status_page = parse_html('portal_status.html')
metrics = {item.get('data-name'): clean_int(item.get_text(' ', strip=True)) for item in status_page.select('.metric')}
print(metrics)

rules = load_json('quality_rules.json')
print(rules['min_students_for_report'])

ready_courses = []
for row in course_rows:
    if status_ok(row['status']) and row['students'] >= rules['min_students_for_report']:
        ready_courses.append(row)
print(ready_courses)

keys = [make_key(row['course'], row['filename']) for row in manifest]
print(len(keys), len(set(keys)))

report_rows = []
for row in ready_courses:
    files = [item for item in manifest if item['course'] == row['course']]
    report_rows.append({'course': row['course'], 'teacher': row['teacher'], 'students': row['students'], 'file_count': len(files), 'status': row['status']})
print(report_rows)

write_csv('lesson10_portal_report.csv', report_rows)
summary = {'notice_count': len(notices), 'course_count': len(course_rows), 'ready_course_count': len(report_rows), 'file_count': len(manifest)}
Path('lesson10_portal_report.json').write_text(json.dumps(summary, ensure_ascii=False, indent=2), encoding='utf-8')
print(summary)

conn = sqlite3.connect('lesson10_portal.db')
conn.execute('drop table if exists report')
conn.execute('create table report(course text, teacher text, students integer, file_count integer, status text)')
conn.executemany('insert into report values(:course, :teacher, :students, :file_count, :status)', report_rows)
print(conn.execute('select course, file_count from report order by students desc').fetchall())
conn.close()

memo = [
    f"공지 {len(notices)}건, 과정 {len(course_rows)}건을 확인했습니다.",
    f"리포트 대상 과정은 {len(report_rows)}건이며 자료 파일은 {len(manifest)}개입니다.",
    f"최근 실행 수는 {metrics.get('runs', 0)}회이고 오류 수는 {metrics.get('errors', 0)}회입니다.",
]
print('\n'.join(memo))
~~~

## 채점 메모

- 입력 파일을 모두 읽었는지 확인한다.
- 검증 기준이 코드에 명시되어 있는지 확인한다.
- 저장 파일과 운영 요약이 함께 있는지 확인한다.

## 채점 보충 기준

학생 답안은 모범 답안과 코드 줄이 완전히 같을 필요는 없다. 다만 포털 링크 수집, 공지/과정/manifest/status/rules 읽기, 리포트 대상 필터링, 저장, 운영 메모가 모두 있어야 한다. 특히 최종 미션은 “실행됐다”보다 “다음 주에도 같은 절차로 다시 실행할 수 있다”가 핵심 기준이다. 파일명과 행 수를 출력하지 않은 답안은 저장 검증이 약한 것으로 본다.
