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

# 레슨 03 — 최종 미션 모범 답안

[![Open in Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/Kevin-innovation/jupyter-lecture/blob/main/python-web-automation/lectures/03/%EB%A0%88%EC%8A%A8%2003%20%E2%80%94%20HTML%20%ED%85%8C%EC%9D%B4%EB%B8%94%EA%B3%BC%20%EB%A6%AC%EC%8A%A4%ED%8A%B8%20%EB%8D%B0%EC%9D%B4%ED%84%B0%20%EC%A0%95%EB%A6%AC.ipynb)

대시보드 표, 피드백 카드, todo 리스트, 루브릭 CSV를 읽어 운영 요약 CSV와 JSON을 만든다.

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

## 모범 답안

~~~python
dashboard_soup = read_soup('class_dashboard.html')
headers = [th.text.strip() for th in dashboard_soup.select('#class-table thead th')]
records = []
for tr in dashboard_soup.select('#class-table tbody tr'):
    row = dict(zip(headers, [td.text.strip() for td in tr.select('td')]))
    row['progress_num'] = percent_to_int(row['progress'])
    row['minutes_num'] = clean_int(row['minutes'])
    records.append(row)

course_summary = defaultdict(lambda: {'count': 0, 'progress_total': 0, 'minutes_total': 0, 'watch': 0})
for row in records:
    bucket = course_summary[row['course']]
    bucket['count'] += 1
    bucket['progress_total'] += row['progress_num']
    bucket['minutes_total'] += row['minutes_num']
    if row['status'] == 'watch':
        bucket['watch'] += 1

summary_rows = []
for course, values in sorted(course_summary.items()):
    summary_rows.append({
        'course': course,
        'student_count': values['count'],
        'avg_progress': round(values['progress_total'] / values['count'], 1),
        'avg_minutes': round(values['minutes_total'] / values['count'], 1),
        'watch_count': values['watch'],
    })

feedback_soup = read_soup('feedback_cards.html')
high_titles = [card.select_one('.title').text.strip() for card in feedback_soup.select('article.feedback-card') if card['data-priority'] == 'high']

todo_soup = read_soup('todo_list.html')
todo_counts = Counter(item['data-status'] for item in todo_soup.select('li.todo-item'))

rubrics = list(csv.DictReader(load_text('rubric.csv').splitlines()))
rubric_total = sum(int(row['max_score']) for row in rubrics)

with open('lesson03_final_summary.csv', 'w', newline='', encoding='utf-8') as f:
    writer = csv.DictWriter(f, fieldnames=['course', 'student_count', 'avg_progress', 'avg_minutes', 'watch_count'])
    writer.writeheader()
    writer.writerows(summary_rows)

operation_note = {
    'student_rows': len(records),
    'high_feedback_count': len(high_titles),
    'high_feedback_titles': high_titles,
    'todo_status': dict(todo_counts),
    'rubric_total': rubric_total,
}
Path('lesson03_final_note.json').write_text(json.dumps(operation_note, ensure_ascii=False, indent=2), encoding='utf-8')
print('saved:', 'lesson03_final_summary.csv', len(summary_rows))
print(operation_note)
~~~

## 왜 이 풀이가 기준 답안인지

표, 카드, 리스트, CSV를 각각 다른 방식으로 읽되 최종 결과는 운영자가 볼 수 있는 요약으로 통일했다. 숫자로 계산할 값은 모두 변환했고, 저장 파일에는 코스별 학생 수와 평균 진도, 관찰 학생 수가 들어간다. high priority 피드백과 todo 상태 분포도 JSON에 남겨 다음 보고서 단계에서 바로 재사용할 수 있다.

## 채점 메모

- records 길이가 원본 table 행 수와 같은지 확인한다.
- course_summary의 count 합계가 records 길이와 같은지 확인한다.
- watch_count는 status가 watch인 행만 세야 한다.
- rubric_total은 100이 되어야 한다.
- 실제 사이트로 확장할 때 요청 간격과 개인정보 확인 기준을 설명할 수 있어야 한다.
