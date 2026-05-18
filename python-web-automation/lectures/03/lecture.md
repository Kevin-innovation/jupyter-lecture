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

# 레슨 03 — HTML 테이블과 리스트 데이터 정리

[![Open in Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/Kevin-innovation/jupyter-lecture/blob/main/python-web-automation/lectures/03/%5B%ED%95%99%EC%83%9D%EC%9A%A9%5D%20%EB%A0%88%EC%8A%A8%2003%20%E2%80%94%20HTML%20%ED%85%8C%EC%9D%B4%EB%B8%94%EA%B3%BC%20%EB%A6%AC%EC%8A%A4%ED%8A%B8%20%EB%8D%B0%EC%9D%B4%ED%84%B0%20%EC%A0%95%EB%A6%AC.ipynb)

이 레슨은 웹 페이지에서 가장 자주 만나는 세 가지 반복 구조를 다룬다. 첫째는 행과 열이 분명한 table, 둘째는 화면 카드처럼 생긴 article, 셋째는 단순 목록인 ul/li이다. 학생은 같은 데이터를 눈으로 보는 방식에서 벗어나, 파이썬이 다루기 쉬운 딕셔너리와 리스트로 바꾸는 과정을 연습한다.

수업 데이터는 모두 합성 fixture다. 실제 학생 개인정보나 외부 사이트 데이터를 쓰지 않는다. 반복 요청을 보내지 않아도 구조를 충분히 연습할 수 있도록 class_dashboard.html, feedback_cards.html, todo_list.html, rubric.csv를 사용한다.

## 학습 목표

1. table의 thead, tbody, tr, th, td 역할을 구분한다.
2. 헤더와 셀을 zip으로 묶어 행 단위 딕셔너리를 만든다.
3. 카드형 UI와 리스트형 UI에서 반복 단위를 찾는다.
4. 퍼센트, 횟수, 분 같은 문자열 값을 숫자로 바꾼다.
5. 여러 출처에서 뽑은 값을 하나의 운영 요약으로 저장한다.

---

## 1. 환경 준비와 데이터 위치 확인

코랩에서는 GitHub raw URL에서 fixture를 읽고, 로컬에서는 현재 레슨 폴더의 data 디렉터리를 읽는다. 이 차이를 한 함수로 감추면 같은 노트북을 코랩과 로컬에서 모두 실행할 수 있다. 먼저 DATA_BASE가 어디를 가리키는지 확인한다.

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

수업에서 가장 먼저 확인할 것은 실행 환경이다. 파일을 못 읽는 오류는 selector 문제가 아니라 경로 문제인 경우가 많다. DATA_BASE 출력이 예상과 다르면 다음 셀로 넘어가기 전에 경로부터 고친다.

---

## 2. table은 헤더와 행을 따로 읽는다

HTML table은 화면에서는 표처럼 보이지만, 코드에서는 thead의 th와 tbody의 tr을 따로 읽어야 한다. class_dashboard.html에는 수업 운영용 학생 현황 표가 들어 있다.

~~~python
dashboard_soup = read_soup('class_dashboard.html')
heading = dashboard_soup.select_one('h1').text.strip()
headers = [th.text.strip() for th in dashboard_soup.select('#class-table thead th')]
rows = dashboard_soup.select('#class-table tbody tr')
print(heading)
print(headers)
print('row count:', len(rows))
~~~

헤더는 나중에 딕셔너리 key가 된다. key를 직접 적어도 되지만, 실제 사이트에서는 컬럼 순서가 바뀌거나 컬럼이 추가될 수 있으므로 화면의 th를 기준으로 읽는 편이 더 안전하다.

### 운영 메모

- selector는 좁게 잡는다. table 전체가 아니라 #class-table thead th처럼 목적 위치를 드러낸다.
- 행 개수는 항상 먼저 확인한다. 첫 행만 출력하면 일부 누락을 발견하지 못한다.
- 헤더 문자열은 공백을 제거한 뒤 저장한다. 공백이 섞이면 딕셔너리 key가 달라진다.

---

## 3. 한 행을 딕셔너리로 바꾸기

tr 하나에는 td가 여러 개 들어 있다. td 텍스트를 리스트로 만들고, headers와 같은 순서로 묶으면 한 학생의 기록이 딕셔너리가 된다.

~~~python
first_row = rows[0]
first_cells = [td.text.strip() for td in first_row.select('td')]
first_record = dict(zip(headers, first_cells))
print(first_record)
~~~

이 방식은 단순하지만 강력하다. 표의 column 이름이 명확할 때는 각 td를 인덱스로 하나씩 꺼내는 것보다 실수가 적다. 학생이 만든 답안을 볼 때도 record에 어떤 key가 있는지 먼저 확인하면 다음 단계의 오류를 빠르게 찾을 수 있다.

---

## 4. 전체 행을 리스트로 정리하기

자동화 결과는 보통 한 행에서 끝나지 않는다. 모든 tr을 순회하며 같은 형태의 딕셔너리를 누적한다. 여기서는 progress, submissions, passed, minutes처럼 숫자로 계산할 값도 같이 변환한다.

~~~python
records = []
for tr in rows:
    cells = [td.text.strip() for td in tr.select('td')]
    row = dict(zip(headers, cells))
    row['progress_num'] = percent_to_int(row['progress'])
    row['submissions_num'] = clean_int(row['submissions'])
    row['passed_num'] = clean_int(row['passed'])
    row['minutes_num'] = clean_int(row['minutes'])
    records.append(row)

print(records[0])
print('records:', len(records))
~~~

문자열 상태 그대로 두면 정렬과 비교가 흔들린다. 예를 들어 '94%'와 '8%'를 문자열로 비교하면 숫자 비교와 다른 결과가 나올 수 있다. 계산에 쓸 값은 별도 필드로 숫자화하는 습관을 들인다.

---

## 5. 상태와 진도 기준으로 필터링하기

운영 대시보드는 전체 목록보다 조건에 맞는 학생을 찾는 데 의미가 있다. active 학생 중 진도율이 80 이상인 학생, watch 상태인 학생, 제출 수가 많은 학생을 따로 뽑아 보자.

~~~python
active_high = [row['student'] for row in records if row['status'] == 'active' and row['progress_num'] >= 80]
watch_students = [row['student'] for row in records if row['status'] == 'watch']
heavy_submitters = [row['student'] for row in records if row['submissions_num'] >= 24]
print('active high:', active_high)
print('watch:', watch_students)
print('heavy submitters:', heavy_submitters)
~~~

필터 조건은 수업 운영 기준과 연결된다. active는 정상 진행, watch는 관찰 필요로 볼 수 있다. 기준을 코드에 적을 때는 숫자를 왜 그렇게 잡았는지 설명할 수 있어야 한다.

---

## 6. 코스별 요약 만들기

여러 학생을 코스별로 묶으면 수업 운영자가 한눈에 상태를 볼 수 있다. defaultdict를 사용하면 코스별 total, count, watch 값을 쉽게 누적할 수 있다.

~~~python
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
        'students': values['count'],
        'avg_progress': round(values['progress_total'] / values['count'], 1),
        'avg_minutes': round(values['minutes_total'] / values['count'], 1),
        'watch_count': values['watch'],
    })
print(summary_rows)
~~~

요약 테이블은 원본 데이터를 줄여서 보여준다. 단순히 줄이는 것이 아니라 수업 판단에 필요한 질문에 답해야 한다. 어떤 코스에 관찰 학생이 많은지, 평균 진도는 어느 정도인지, 학습 시간이 낮은 코스는 없는지 확인한다.

---

## 7. 카드형 UI 읽기

feedback_cards.html은 표가 아니라 article.feedback-card가 반복되는 구조다. 표처럼 헤더가 없기 때문에 카드 하나에서 어떤 selector와 속성을 읽을지 직접 정해야 한다.

~~~python
feedback_soup = read_soup('feedback_cards.html')
feedback_cards = feedback_soup.select('article.feedback-card')
feedback_records = []
for card in feedback_cards:
    feedback_records.append({
        'title': card.select_one('.title').text.strip(),
        'summary': card.select_one('.summary').text.strip(),
        'priority': card['data-priority'],
        'index': int(card['data-index']),
    })
print(feedback_records[:3])
~~~

카드형 UI에서는 반복 단위를 잘못 잡는 실수가 잦다. section을 반복하면 카드가 하나로 뭉치고, h2만 반복하면 summary와 priority를 잃는다. 반복 단위는 하나의 완성된 데이터 객체를 담는 태그여야 한다.

---

## 8. 우선순위 카드 추출하기

card의 data-priority 속성은 화면에 보이지 않을 수 있지만 자동화에는 유용하다. high, normal, low를 기준으로 개수를 세고, high 카드 제목만 따로 모은다.

~~~python
priority_counts = Counter(card['priority'] for card in feedback_records)
high_titles = [card['title'] for card in feedback_records if card['priority'] == 'high']
print(priority_counts)
print(high_titles)
~~~

속성 값은 텍스트보다 안정적인 경우가 많다. 다만 사이트가 바뀌면 속성 이름이 바뀔 수 있으므로, selector와 속성 이름을 수업 노트에 같이 남겨 둔다.

---

## 9. 리스트형 UI 읽기

todo_list.html은 li.todo-item이 반복된다. 리스트는 구조가 단순하지만, 상태가 data-status 속성에 들어 있으므로 텍스트만 읽으면 중요한 정보를 놓친다.

~~~python
todo_soup = read_soup('todo_list.html')
todo_items = []
for item in todo_soup.select('li.todo-item'):
    todo_items.append({
        'order': int(item['data-order']),
        'status': item['data-status'],
        'text': item.text.strip(),
    })
print(todo_items[:4])
print(Counter(item['status'] for item in todo_items))
~~~

리스트 데이터는 간단해 보이지만 순서가 중요한 경우가 많다. data-order를 숫자로 바꾸어 두면 나중에 정렬하거나 누락 번호를 찾을 수 있다.

---

## 10. 평가 기준 CSV 읽기

rubric.csv는 최종 산출물 평가 기준을 담고 있다. HTML만 다루는 것이 아니라 CSV를 함께 읽어야 실제 자동화 흐름에서 입력과 출력 형식을 연결할 수 있다.

~~~python
rubrics = list(csv.DictReader(load_text('rubric.csv').splitlines()))
for row in rubrics:
    row['max_score'] = int(row['max_score'])
print(rubrics)
print('total score:', sum(row['max_score'] for row in rubrics))
~~~

CSV를 읽을 때 숫자 컬럼도 문자열로 들어온다. max_score를 합산하려면 int로 변환해야 한다. 웹에서 읽은 값과 파일에서 읽은 값 모두 타입 확인이 필요하다.

---

## 11. 운영 요약 저장하기

마지막으로 여러 구조에서 뽑은 데이터를 하나의 요약으로 저장한다. 여기서는 코스별 요약은 CSV로 저장하고, 카드와 할 일 상태는 JSON으로 저장한다.

~~~python
with open('lesson03_course_summary.csv', 'w', newline='', encoding='utf-8') as f:
    writer = csv.DictWriter(f, fieldnames=['course', 'students', 'avg_progress', 'avg_minutes', 'watch_count'])
    writer.writeheader()
    writer.writerows(summary_rows)

operation_summary = {
    'student_rows': len(records),
    'courses': len(summary_rows),
    'high_feedback_titles': high_titles,
    'todo_status': dict(Counter(item['status'] for item in todo_items)),
    'rubric_total': sum(row['max_score'] for row in rubrics),
}
Path('lesson03_operation_summary.json').write_text(json.dumps(operation_summary, ensure_ascii=False, indent=2), encoding='utf-8')
print('saved csv rows:', len(summary_rows))
print(operation_summary)
~~~

저장 파일은 다음 자동화 단계의 입력이 된다. 그래서 컬럼 이름과 JSON key를 사람이 이해할 수 있게 정해야 한다. 저장 전에는 개수, 평균, 상태 분포가 원본과 맞는지 간단히 확인한다.

---

## 수업 정리

이번 레슨의 핵심은 화면 모양이 아니라 반복 단위를 찾는 것이다. table은 tr, 카드 UI는 article.feedback-card, 목록은 li.todo-item이 반복 단위였다. 반복 단위를 찾은 뒤에는 텍스트와 속성을 분리해서 읽고, 계산에 쓸 값은 숫자로 변환했다.

실제 사이트로 확장할 때는 다음 기준을 적용한다.

- 수집 대상이 약관과 robots.txt에 어긋나지 않는지 확인한다.
- 개인정보나 로그인 후 데이터는 수업 예제로 사용하지 않는다.
- 요청 간격과 최대 요청 수를 코드에 둔다.
- 저장 파일에는 수집 시각, 출처, 필터 기준을 남긴다.
- selector가 깨졌을 때 빈 결과를 성공으로 처리하지 않는다.

3강 이후부터는 이렇게 정리한 데이터에 오류 검증, 파일 저장, 반복 실행 기준을 더해 운영 자동화 형태로 확장한다.

---

## 12. selector 디버깅 루틴

selector가 틀리면 대부분 NoneType 오류가 나거나 빈 리스트가 나온다. 이때 바로 정답 selector를 외우게 하면 다음 구조에서 다시 막힌다. 수업에서는 아래 순서로 학생이 스스로 좁혀 가게 한다.

1. 파일이 열렸는지 확인한다. load_text 결과의 앞부분을 짧게 출력해 HTML이 실제로 들어왔는지 본다.
2. 가장 넓은 태그부터 확인한다. table, article, li처럼 큰 반복 단위가 몇 개인지 센다.
3. id와 class를 붙여 좁힌다. #class-table tbody tr, article.feedback-card, li.todo-item처럼 목적이 드러나는 selector를 쓴다.
4. 하나의 반복 단위 안에서 하위 selector를 찾는다. card.select_one('.title')처럼 전체 soup가 아니라 현재 card 안에서 찾는다.
5. 최종 결과 개수와 원본 개수를 비교한다. 원본 행 수와 records 길이가 다르면 반복 범위가 잘못된 것이다.

~~~python
raw = load_text('class_dashboard.html')
print(raw[:80])
print('table:', len(BeautifulSoup(raw, 'html.parser').select('table')))
print('rows:', len(BeautifulSoup(raw, 'html.parser').select('#class-table tbody tr')))
~~~

이 루틴은 실제 사이트에서도 그대로 쓸 수 있다. 단, 실제 사이트에서는 요청 횟수를 줄이기 위해 HTML을 한 번 받아 변수에 저장하고, 그 변수에서 여러 selector를 테스트한다. 같은 URL을 계속 새로 요청하는 방식은 수업에서도 운영에서도 피한다.

---

## 13. 데이터 품질 검토

자동화 결과가 맞는지 보려면 출력값 하나보다 품질 기준을 확인해야 한다. 이번 레슨에서는 다음 네 가지를 본다.

| 기준 | 확인 방법 | 문제가 있을 때 |
|---|---|---|
| 행 개수 | len(records)가 원본 tr 개수와 같은가 | selector가 너무 넓거나 좁다 |
| key 일관성 | 모든 record가 같은 key를 갖는가 | headers와 cells 길이가 다르다 |
| 타입 안정성 | progress_num, minutes_num이 int인가 | 문자열 비교 오류가 생긴다 |
| 저장 가능성 | CSV fieldnames와 row key가 맞는가 | 저장 파일이 비거나 컬럼이 밀린다 |

~~~python
print('records:', len(records), 'rows:', len(rows))
print('keys:', sorted(records[0].keys()))
print('progress type:', type(records[0]['progress_num']).__name__)
print('minutes type:', type(records[0]['minutes_num']).__name__)
~~~

학생 답안 검수에서는 완성 코드가 정답과 똑같은지보다 위 네 기준을 만족하는지 보는 편이 좋다. 특히 selector는 여러 방식이 가능하다. 결과 개수와 key, 타입, 저장 산출물이 맞으면 인정할 수 있다.

---

## 14. 운영 보고서로 연결하기

수집한 데이터는 보고서 문장으로 바꿔야 의미가 생긴다. 코스별 평균 진도, 관찰 학생 수, 긴급 피드백 수, pending todo 수는 선생님이 바로 판단할 수 있는 지표다.

~~~python
report_lines = []
for row in summary_rows:
    report_lines.append(
        f"{row['course']}: 학생 {row['students']}명, 평균 진도 {row['avg_progress']}%, 관찰 {row['watch_count']}명"
    )
report_lines.append(f"긴급 피드백 {len(high_titles)}건, pending todo {Counter(item['status'] for item in todo_items).get('pending', 0)}건")
print('\n'.join(report_lines))
~~~

보고 문장을 만들면 학생은 자동화가 왜 필요한지 이해한다. 단순히 HTML을 긁어오는 것이 목표가 아니라, 반복 작업을 줄이고 운영자가 볼 수 있는 정보로 정리하는 것이 목표다.

---

## 15. 실제 사이트 확장 전 점검

이 레슨의 fixture는 안전한 합성 데이터지만, 실제 사이트로 확장할 때는 기준이 더 엄격하다. 로그인 후 화면, 학생 정보, 유료 콘텐츠, 외부 서비스 데이터는 허가 없이 수집하면 안 된다. 또한 같은 페이지를 짧은 간격으로 반복 요청하면 서비스에 부담을 줄 수 있다.

실제 확장 전에는 다음 항목을 문서에 남긴다.

- 수집 목적: 어떤 업무를 줄이기 위한 자동화인가.
- 수집 범위: 어떤 페이지와 어떤 필드만 읽는가.
- 요청 제한: 최대 페이지 수, 요청 간격, 재시도 횟수는 얼마인가.
- 개인정보 여부: 이름, 연락처, 학습 기록이 들어가는가.
- 저장 위치: CSV, JSON, DB 중 어디에 저장하고 누가 접근하는가.

이 기준을 통과하지 못하면 코드를 작성하지 않는다. 학생에게도 이 기준을 반복해서 알려야 웹 자동화가 단순한 크롤링이 아니라 책임 있는 운영 도구라는 점이 잡힌다.

