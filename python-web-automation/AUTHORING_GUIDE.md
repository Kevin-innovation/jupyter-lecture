# 작성 가이드 — python-web-automation

이 문서는 새 레슨을 만들 때 따라야 하는 형식과 변환 흐름을 정의한다. `python-data-analysis` 코스의 제작 프로세스, 문제 수 기준, 교사용 정답 해설 기준, 테스트 검증 흐름을 웹 자동화 코스에 그대로 이식한 기준 문서다. AI(Claude 등) 보조 작성 시에도 이 문서 규칙을 우선한다.

## 1. 원본은 마크다운 (`.md`), 학생 노트북은 자동 생성 (`.ipynb`)

- 사람이 직접 손보는 파일은 항상 `.md`다.
- `.ipynb` 는 `jupytext` 로 변환해서 만든다. 원칙적으로 `.ipynb` 를 직접 수정하지 않는다.
- 둘 다 git 에 커밋한다. 학생은 앱에서 통합 `.ipynb` 를 열고, 제작자는 분리 `.md` 파일을 기준으로 유지보수한다.
- 앱 Colab 버튼이 여는 통합 노트북 2개는 분리 파일을 합쳐 만든다.

### 설치

```bash
pip install jupytext
```

### 변환

레슨 폴더에서:

```bash
jupytext --to ipynb lecture.md
jupytext --to ipynb mission.md
jupytext --to ipynb solution.md
jupytext --to ipynb final-mission.md
jupytext --to ipynb final-mission-solution.md
```

또는 일괄:

```bash
cd docs/courses/active/python-web-automation
find lectures -name "*.md" ! -name "teacher_guide.md" ! -name "README.md" -exec jupytext --to ipynb {} \;
```

`teacher_guide.md`, 코스 루트의 `CURRICULUM.md` 등은 변환 대상이 아니다.

## 2. 마크다운 셀 표기 규칙

jupytext 는 마크다운에서 코드 블록을 보면 코드 셀로, 일반 텍스트는 마크다운 셀로 만든다. 셀 구분이 모호하면 빈 줄을 넣어 명확히 한다.

### 마크다운 셀

```markdown
## 2-1. HTML을 데이터로 보는 법

브라우저 화면은 사람이 보기 좋게 렌더링된 결과다.
파이썬은 화면 대신 HTML 문자열을 읽고 태그 구조를 탐색한다.
```

### 코드 셀

````markdown
```python
from bs4 import BeautifulSoup

soup = BeautifulSoup(html, "html.parser")
print(soup.select_one("h1").text.strip())
```
````

코드 블록 언어는 반드시 `python` 으로 명시한다. `python` 외 언어는 셀 변환 대상이 아니다.

### 셀 출력은 적지 않는다

- 마크다운에 코드 실행 결과(`상품 카드 수: 8` 같은 출력값)를 직접 적지 않는다.
- 출력 형태를 알려야 하면 "이런 형태로 출력됩니다" 정도만 쓴다.
- `.ipynb` 변환 시 셀 출력은 비어 있는 상태로 만든다. 학생이 직접 실행해서 채운다.

## 3. 레슨별 파일 표준

```
lectures/<NN>/
├── [학생용] 레슨 NN — 제목.ipynb
├── 레슨 NN — 제목.ipynb
├── lecture.md
├── lecture.ipynb
├── mission.md
├── mission.ipynb
├── solution.md
├── solution.ipynb
├── final-mission.md
├── final-mission.ipynb
├── final-mission-solution.md
├── final-mission-solution.ipynb
├── teacher_guide.md
└── data/
```

각 파일의 책임은 다음과 같다.

### `[학생용] 레슨 NN — 제목.ipynb`

- 앱 상단 Colab 버튼이 여는 학생용 통합 노트북.
- `lecture` + `mission` + `final-mission` 을 한 흐름으로 묶는다.
- 정답 코드, 교사용 해설, "교사·관리자 전용" 문구가 들어가면 실패다.
- 첫 마크다운 셀에는 Colab 배지가 있어야 한다.

### `레슨 NN — 제목.ipynb`

- 앱 상단 선생님용 Colab 버튼이 여는 교사용 통합 노트북.
- `lecture` + `solution` + `final-mission-solution` + 수업 운영 메모를 묶는다.
- 모든 문제 정답 아래에 `왜 이 코드가 정답인지` 설명이 있어야 한다.

### `lecture.md`

학생이 가장 처음 읽는 강의 원본. 다음 5절 구조를 지킨다.

1. **학습 목표** — 3~5개의 측정 가능한 목표.
2. **개념 설명** — HTTP, HTML, selector, 브라우저 자동화 개념을 코드보다 글로 먼저 설명.
3. **현업·운영 활용 사례** — 업무 자동화, 공개 데이터 수집, QA 자동화 등 구체 사례 1개 이상.
4. **예제 코드 (따라하기)** — 셀 단위로 끊어진 실행 가능한 코드. 5~10셀.
5. **데이터 출처와 안전 규칙** — 합성 데이터 여부, robots.txt, 요청 간격, 금지 행위.

### `mission.md`

- 학생용 실습 문제는 **최소 15개, 최대 20개**로 작성한다. 기본 권장값은 **15문제**다.
- 문제 heading 은 반드시 `## 문제 1 — ...`, `## 문제 2 — ...` 형식으로 시작하고, 1번부터 빠짐없이 연속 넘버링한다.
- 권장 난이도 배치는 1~5번 기초 확인, 6~10번 응용 연습, 11~15번 실전/해석형이다. 16~20번은 빠른 학생용 선택 심화로만 둔다.
- 각 문제는 다음 4요소를 포함한다: 문제 설명 / 입력 데이터 / 기대 결과(형태만, 정답값 X) / 학생이 채울 코드 셀.
- 힌트는 정답 함수 호출이나 완성 코드를 그대로 노출하지 않는다.
- 학생용 코드 셀에는 `____` 빈칸 scaffold를 둔다.
- 정답값은 절대 적지 않는다. 정답값과 모범 코드는 `solution.md` 에만 둔다.
- 최종 프로젝트인 `final-mission.md` 는 15~20문제 수에 포함하지 않는다.

### `solution.md`

- `mission.md` 의 모든 문제와 동일한 번호로 정답을 제공한다. heading 은 `## 문제 1 정답 — ...` 형식을 쓴다.
- 학생용 문제가 15개면 정답도 15개여야 한다. 문제 수와 정답 수가 다르면 검증 실패다.
- 각 정답 코드 아래에는 반드시 `### 왜 이 코드가 정답인지` 마크다운 셀을 두고, 코드가 요구사항을 어떻게 충족하는지 자세히 설명한다.
- 설명에는 사용한 함수/메서드의 역할, HTML selector 기준, 데이터 컬럼 또는 태그 구조, 정답 판단 기준, 자주 틀리는 포인트를 포함한다.
- 기대 출력이나 채점 포인트가 필요하면 `### 기대 출력`, `### 채점 포인트`, `### 자주 보이는 오답` 절을 추가한다.
- 파일 상단에 큼지막하게 "교사·관리자 전용. 학생 배포 금지." 명시.

### `final-mission.md`

- 레슨 내용을 통합하는 단일 프로젝트 미션.
- 데이터, 요구사항(필수 5개 + 보너스 2개), 산출물 형태(CSV, 표, 짧은 결론)를 명시.
- 마지막 셀은 "자동화 결과 요약" 작성용 빈 마크다운 셀로 남긴다.

### `final-mission-solution.md`

- 모범 답안 노트북. 학생 비공개.
- 실행 가능한 전체 코드와 채점 메모를 포함한다.

### `teacher_guide.md`

- 이 레슨용 교사 가이드. 노트북 변환 대상 아님.
- 표준 항목은 아래 §6 참조.

### `data/`

- 이 레슨에서만 쓰는 HTML/CSV/TXT 파일.
- 1~5강은 실제 사이트 대신 합성 HTML 데이터를 우선 사용한다.
- 파일 헤더에 컬럼 의미가 한눈에 보이도록 한국어 헤더는 피하고 영어 snake_case 컬럼명 사용.

## 4. 데이터와 사이트 fixture 규칙

- 첫 5강은 `data/*.html`, `data/*.csv`, `data/*.txt` 로 재현 가능한 fixture를 만든다.
- 외부 사이트를 직접 때리는 코드는 강사용 시연 또는 후반부 레슨에서만 제한적으로 사용한다.
- 합성 HTML에는 실제 사이트를 연상시키는 개인정보, 실명, 실제 주문번호를 넣지 않는다.
- `data/README.md` 에 각 파일의 목적, 생성 방식, 연습할 selector를 적는다.
- CSV 파일은 UTF-8, 헤더 포함, 행 수 30~500행을 기준으로 한다.
- HTML fixture는 학생이 구조를 읽을 수 있게 class/id 이름을 의미 있게 둔다.
- robots.txt 예시는 수업용 샘플로 제공하고, 실제 robots 해석과 동일하다고 과장하지 않는다.

## 5. 코랩 / VSCode 양쪽 호환 노트북 시작 셀

모든 레슨 첫 코드 셀에는 환경 감지 + 데이터 경로 분기 + 안전한 로드 헬퍼를 둔다.

```python
import os
import re
import time
import csv
from pathlib import Path
from urllib.parse import urljoin

try:
    import requests
    from bs4 import BeautifulSoup
except ImportError:
    import sys, subprocess
    subprocess.check_call([sys.executable, "-m", "pip", "-q", "install", "requests", "beautifulsoup4"])
    import requests
    from bs4 import BeautifulSoup

IS_COLAB = "COLAB_GPU" in os.environ or "COLAB_TPU_ADDR" in os.environ

if IS_COLAB:
    DATA_BASE = "https://raw.githubusercontent.com/Kevin-innovation/jupyter-lecture/main/python-web-automation/lectures/<NN>/data"
else:
    DATA_BASE = "./data"

def load_text(filename):
    if DATA_BASE.startswith("http"):
        url = f"{DATA_BASE}/{filename}"
        response = requests.get(url, timeout=10, headers={"User-Agent": "D-Lab-Lesson/1.0"})
        response.raise_for_status()
        response.encoding = response.encoding or "utf-8"
        return response.text
    return Path(DATA_BASE, filename).read_text(encoding="utf-8")

def clean_int(text):
    return int(re.sub(r"[^0-9]", "", text))

print("data base:", DATA_BASE)
```

`<NN>` 은 실제 레슨 번호로 교체한다. 저장소 비공개라면 코랩에서는 학생이 직접 업로드하는 흐름으로 안내한다.

## 6. `teacher_guide.md` 표준 항목

각 레슨 교사 가이드는 다음 8개 절을 반드시 포함한다.

```markdown
# 레슨 NN — 교사 가이드

## 학습 목표 (학생용보다 더 상세)
## 2시간 타임라인 (분 단위)
## 사전 준비물
## 핵심 개념 강조 포인트 (수업 진행 시 강조할 부분)
## 학생이 자주 막히는 지점과 대처법
## 실습 문제 채점 포인트 (문제별)
## 최종 미션 평가 루브릭 (필수 / 보너스 / 감점)
## 다음 레슨과의 연결고리
```

## 7. 작성 순서 권장

새 레슨을 만들 때 권장 순서:

1. `teacher_guide.md` 의 학습 목표·타임라인 먼저.
2. `data/` 의 실습 HTML/CSV/robots fixture 준비.
3. `lecture.md` 의 개념 설명 + 예제 코드.
4. `mission.md` 의 문제 15~20개(기본 15개).
5. `solution.md` 채우기.
6. `final-mission.md` + `final-mission-solution.md`.
7. `jupytext --to ipynb` 일괄 변환.
8. 학생용/선생님용 통합 노트북 2개 생성.
9. 코랩에서 실제로 한 번 끝까지 실행해보기. 의도대로 동작하지 않으면 마크다운 수정 후 재변환.
10. **§8 파일 용량 제한과 §10 PR 체크리스트**를 통과하는지 검증.

## 8. 파일 용량 제한 (필수 검증)

레슨 산출물의 분량은 `python-data-analysis` 와 같은 기준을 쓴다. 너무 작으면 내용이 부실하고, 너무 크면 학생이 한 회차에 소화하지 못한다. **PR 머지 전에 반드시 검증한다.**

### 8-1. 파일별 용량 표

| 파일 | 최소 | 최대 | 의도 |
|---|---:|---:|---|
| `lecture.md` | **18 KB** | **24 KB** | 2시간 강의의 본문. 개념 + 안전 규칙 + 현업 사례 + 예제 코드 + 따라하기 + 실수 정리 |
| `mission.md` | 14 KB | 24 KB | 15~20개 학생 문제 + 힌트 + 통과 기준 |
| `solution.md` | 24 KB | 42 KB | 문제별 정답 + 왜 정답인지 설명 + 기대 출력 + 채점 포인트 + 오답 패턴 |
| `final-mission.md` | 5 KB | 8 KB | 시나리오 + 필수/보너스 + 진행 가이드 + 평가 루브릭 |
| `final-mission-solution.md` | 5 KB | 9 KB | 모범 답안 전체 + 채점 메모 |
| `teacher_guide.md` | 5 KB | 9 KB | §6 표준 8개 절 |
| `data/README.md` | 1 KB | 4 KB | 파일 목록 + fixture 의도 + 생성 방식 |

### 8-2. 데이터 최소 기준

| 파일 유형 | 최소 기준 | 비고 |
|---|---:|---|
| 반복 카드 HTML | 8개 item | selector 반복 추출 연습 가능 |
| 공지/게시판 HTML | 10개 item | 날짜, 제목, 링크, 카테고리 추출 가능 |
| CSV target 목록 | 10행 | URL/파일명/우선순위 반복 처리 가능 |
| robots/sample policy | 5개 규칙 | 허용/차단/지연 개념 설명 가능 |

외부 웹 요청이 필요한 레슨은 실제 요청 횟수를 줄이기 위해 캐시 파일 또는 샘플 응답 파일을 함께 둔다.

### 8-3. 용량 검증 명령

레슨 폴더에서 다음을 실행해 바이트 수를 확인한다.

```bash
cd lectures/<NN>
for f in lecture.md mission.md solution.md final-mission.md final-mission-solution.md teacher_guide.md data/README.md; do
  bytes=$(wc -c < "$f")
  printf "%-32s %6d B\n" "$f" "$bytes"
done
```

코스 루트에서 한도 위반 파일만 골라내는 검증 스크립트:

```bash
cd docs/courses/active/python-web-automation
python3 - <<'PY'
import os, sys

LIMITS = {
    "lecture.md":                (18_000, 24_000),
    "mission.md":                (14_000, 24_000),
    "solution.md":               (24_000, 42_000),
    "final-mission.md":          ( 5_000,  8_000),
    "final-mission-solution.md": ( 5_000,  9_000),
    "teacher_guide.md":          ( 5_000,  9_000),
    "data/README.md":            ( 1_000,  4_000),
}

fail = 0
for root, _, files in os.walk("lectures"):
    for name in files:
        rel = os.path.join(root, name)
        parts = rel.split(os.sep)
        if len(parts) < 3:
            continue
        key = os.path.join(*parts[2:]) if len(parts) > 3 else parts[-1]
        if key not in LIMITS:
            continue
        size = os.path.getsize(rel)
        lo, hi = LIMITS[key]
        status = "OK " if lo <= size <= hi else "BAD"
        if status == "BAD":
            fail += 1
        print(f"{status} {size:>6} B  ({lo:_}-{hi:_})  {rel}")

sys.exit(1 if fail else 0)
PY
```

`BAD` 가 한 줄이라도 보이면 PR 머지 금지. 본문을 늘리거나 줄여 한도 안으로 들어오게 조정한 뒤 다시 돌린다.

### 8-4. 문제/정답/통합 노트북 검증 명령

```bash
cd docs/courses/active/python-web-automation
python3 - <<'PY'
import json
import os
import re
import sys
from pathlib import Path

fail = 0

def text_of_cell(cell):
    src = cell.get("source", "")
    return "".join(src) if isinstance(src, list) else str(src)

for lesson_dir in sorted(Path("lectures").glob("[0-9][0-9]")):
    mission = (lesson_dir / "mission.md").read_text(encoding="utf-8")
    solution = (lesson_dir / "solution.md").read_text(encoding="utf-8")
    mission_nums = [int(n) for n in re.findall(r"^## 문제 (\\d+)\\b", mission, re.M)]
    solution_nums = [int(n) for n in re.findall(r"^## 문제 (\\d+) 정답\\b", solution, re.M)]
    explain_count = len(re.findall(r"^### 왜 이 코드가 정답인지\\b", solution, re.M))

    expected = list(range(1, len(mission_nums) + 1))
    checks = [
        (15 <= len(mission_nums) <= 20, f"{lesson_dir}: mission problem count {len(mission_nums)}"),
        (mission_nums == expected, f"{lesson_dir}: mission numbering {mission_nums}"),
        (solution_nums == mission_nums, f"{lesson_dir}: solution numbering mismatch {solution_nums}"),
        (explain_count == len(mission_nums), f"{lesson_dir}: explanation count {explain_count}"),
        ("____" in mission, f"{lesson_dir}: student scaffold missing ____"),
    ]

    notebooks = [p for p in lesson_dir.glob("*.ipynb") if p.name.startswith("[학생용] 레슨 ")]
    teacher_notebooks = [p for p in lesson_dir.glob("*.ipynb") if p.name.startswith("레슨 ")]
    checks.append((len(notebooks) == 1, f"{lesson_dir}: student integrated notebook count {len(notebooks)}"))
    checks.append((len(teacher_notebooks) == 1, f"{lesson_dir}: teacher integrated notebook count {len(teacher_notebooks)}"))

    for nb_path, role in [(notebooks[0] if notebooks else None, "student"), (teacher_notebooks[0] if teacher_notebooks else None, "teacher")]:
        if not nb_path:
            continue
        nb = json.loads(nb_path.read_text(encoding="utf-8"))
        cells = nb.get("cells", [])
        first = text_of_cell(cells[0]) if cells else ""
        all_text = "\n".join(text_of_cell(c) for c in cells)
        checks.append(("colab-badge.svg" in first, f"{nb_path}: first cell Colab badge present"))
        if role == "student":
            forbidden = ["교사·관리자 전용", "문제 1 정답", "왜 이 코드가 정답인지"]
            checks.append((not any(word in all_text for word in forbidden), f"{nb_path}: no teacher-only text leaked"))

    for ok, msg in checks:
        print(("OK  " if ok else "BAD ") + msg)
        if not ok:
            fail += 1

sys.exit(1 if fail else 0)
PY
```

### 8-5. 한도를 벗어났을 때 처리 방향

- **lecture.md 가 18KB 미달**: HTTP/HTML 개념, selector 실수 사례, 안전 규칙, 현업 사례, 디버깅 체크포인트를 보강한다.
- **mission.md 가 14KB 미달**: 문제 수가 15개 미만인지 먼저 확인. 각 문제의 입력 데이터, 기대 결과 형태, 빈칸 scaffold, 힌트를 보강한다.
- **solution.md 가 24KB 미달**: 각 정답 아래 `왜 이 코드가 정답인지`, selector 기준, 오답 패턴, 채점 포인트를 보강한다.
- **final mission 이 미달**: 시나리오, 산출물 정의, 평가 루브릭, 제출 전 체크를 늘린다.
- **teacher guide 가 미달**: 2시간 타임라인, 문제별 채점 포인트, 막히는 지점별 멘트를 구체화한다.

## 9. 웹 자동화 안전 한스푼 (필수 규칙)

학생 대부분은 웹 요청이 서버에 부하를 줄 수 있다는 감각이 없다. 이 코스는 "된다"보다 "안전하게 된다"를 먼저 가르친다. 다음 개념이 본문에 처음 등장하는 위치에서 짧은 박스를 둔다.

### 9-1. 박스 형식

```markdown
> **🛡️ 웹 자동화 안전 한스푼 — <용어>**
>
> - **뜻**: 한 문장 정의
> - **왜 중요한가**: 서버 부하, 법적·운영 리스크, 디버깅 관점 중 하나
> - **수업 기준**: 이 코스에서 지키는 구체 규칙
> - **실수 예시**: 학생이 흔히 하는 잘못된 코드/생각 한 줄
```

### 9-2. 박스를 둬야 하는 용어

| 용어 | 처음 등장 권장 레슨 |
|---|---|
| HTTP 요청/응답, status_code | 01 |
| timeout, raise_for_status | 01 |
| User-Agent | 01 |
| HTML selector, class/id/attribute | 01 |
| robots.txt | 01 |
| 요청 간격(rate limit) | 01 또는 02 |
| query string | 02 |
| pagination | 02 |
| relative URL / absolute URL | 01 또는 02 |
| session/cookie | 05 |
| browser automation wait | 06 또는 07 |
| retry/backoff/logging | 09 |

새 용어가 본문에 등장하는데 이 표에 없다면 표에 한 줄 추가하면서 박스도 만든다.

### 9-3. 작성 원칙

- 학생이 그대로 따라 할 수 있는 안전 기준을 적는다. 예: `timeout=10`, 요청 사이 `time.sleep(1)`, 무한 루프 금지.
- 실제 사이트 이름이 들어가면 약관/robots 확인 전제도 같이 적는다.
- 금지 예시는 분명하게 적되, 우회 방법은 쓰지 않는다.
- 한 레슨에 박스가 너무 많으면 흐름이 끊긴다. 4~7개를 기준으로 한다.

### 9-4. 검증

PR 머지 전, 다음을 직접 확인한다.

- [ ] §9-2 표에 있는 용어 중 본문에 등장한 모든 용어에 박스가 있다.
- [ ] 박스가 처음 등장 위치에 있다.
- [ ] 같은 용어가 다음 레슨에서 다시 나오면 "→ 레슨 NN 참조" 로 연결된다.
- [ ] 우회, 공격, 개인정보 수집을 실행 가능한 절차로 설명하지 않았다.

## 10. PR 체크리스트 (레슨 추가 시)

```
- [ ] lecture.md 작성 완료 (18~24 KB 범위)
- [ ] mission.md 작성 완료 (14~24 KB, 문제 15~20개, 1번부터 연속 넘버링, 정답값 미포함)
- [ ] solution.md 작성 완료 (24~42 KB, 학생 문제와 동일 번호, 각 정답 아래 `왜 이 코드가 정답인지` 설명 포함, 상단에 비공개 표시)
- [ ] mission.ipynb / solution.ipynb 의 문제 수와 정답 수가 md 원본과 일치
- [ ] mission.ipynb 에 정답값, 모범 답안, 교사용 해설이 노출되지 않음
- [ ] final-mission.md (5~8 KB) / final-mission-solution.md (5~9 KB) 작성 완료
- [ ] teacher_guide.md 8개 절 모두 채움 (5~9 KB)
- [ ] data/ 의 모든 파일이 UTF-8 + 헤더 포함 또는 HTML 구조 설명 포함
- [ ] data/README.md 가 1~4 KB 범위이고 fixture 의도가 명확함
- [ ] §8-3 용량 검증 스크립트가 "BAD" 없이 통과
- [ ] §8-4 문제/정답/통합 노트북 검증 스크립트가 "BAD" 없이 통과
- [ ] §9 웹 자동화 안전 한스푼 박스가 새로 등장한 모든 안전 개념에 붙어 있음
- [ ] jupytext 변환 결과 코랩에서 처음부터 끝까지 실행 확인
- [ ] 학생용 통합 노트북 첫 셀에 Colab 배지가 있음
- [ ] 선생님용 통합 노트북 첫 셀에 Colab 배지가 있음
- [ ] 라이선스/출처 확인 (실명·개인정보 없음, 합성 fixture 또는 공개 데이터)
- [ ] CURRICULUM.md 의 레슨 개요 표와 일치
- [ ] `.DS_Store` 등 잡파일이 커밋에 포함되지 않았는지 확인
```

## 11. AI 보조 제작 프롬프트 표준

AI에게 새 레슨 제작을 맡길 때는 아래 프롬프트를 그대로 붙이고, `<...>` 만 바꾼다.

```text
너는 D-Lab 파이썬 웹 자동화 코스의 교안 제작자다.
반드시 docs/courses/active/python-web-automation/AUTHORING_GUIDE.md 를 최우선 기준으로 따른다.

대상 레슨:
- 레슨 번호: <NN>
- 제목: <제목>
- 핵심 도구: <requests / BeautifulSoup / urllib.parse / Playwright 등>
- 최종 산출물: <CSV / JSON / 보고서 / 자동화 스크립트>

반드시 지킬 조건:
1. 원본은 md 파일이다. ipynb는 jupytext 변환물이다.
2. lecture.md는 18~24KB, mission.md는 14~24KB, solution.md는 24~42KB다.
3. mission.md 문제는 최소 15개, 최대 20개이며 1번부터 연속 번호다.
4. 학생용 힌트는 정답 코드의 90%를 주지 않는다. 코드 셀에는 ____ 빈칸 scaffold를 둔다.
5. solution.md는 mission.md와 같은 번호의 정답을 모두 포함한다.
6. 각 정답 아래에는 반드시 ### 왜 이 코드가 정답인지 를 쓰고, selector/HTTP/데이터 구조/채점 기준을 설명한다.
7. 학생용 통합 노트북에는 정답, 교사용 해설, 교사·관리자 전용 문구를 넣지 않는다.
8. 선생님용 통합 노트북에는 모든 정답과 최종 미션 모범 답안을 넣는다.
9. 외부 사이트를 무리하게 요청하지 않는다. 필요하면 합성 HTML fixture를 data/에 만든다.
10. robots.txt, timeout, status_code, 요청 간격, User-Agent 같은 안전 개념을 수업 흐름 안에 포함한다.

산출물:
- lectures/<NN>/lecture.md
- lectures/<NN>/mission.md
- lectures/<NN>/solution.md
- lectures/<NN>/final-mission.md
- lectures/<NN>/final-mission-solution.md
- lectures/<NN>/teacher_guide.md
- lectures/<NN>/data/README.md 및 필요한 fixture
- jupytext 변환 ipynb
- [학생용] 레슨 NN — 제목.ipynb
- 레슨 NN — 제목.ipynb

작업 후 반드시 AUTHORING_GUIDE.md §8-3, §8-4 검증을 실행하고 결과를 보고한다.
```
