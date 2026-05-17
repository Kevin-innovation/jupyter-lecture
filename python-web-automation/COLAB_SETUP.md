# 코랩 / VSCode 환경 셋업 — 학생용

이 코스는 **코랩(Google Colab)** 또는 **VSCode + Jupyter** 에서 진행합니다. 우리 사이트의 IDE 가 아닙니다. 어느 쪽을 골라도 결과는 같습니다. 웹 자동화 코스는 외부 사이트를 무리하게 요청하지 않고, 수업용 fixture 파일을 코랩에서 읽어 실습합니다.

## A. 코랩으로 진행하는 방법 (권장 — 설치 없음)

### 1. 코랩 접속

[https://colab.research.google.com](https://colab.research.google.com) 에 접속해 구글 계정으로 로그인합니다.

### 2. 노트북 열기

세 가지 방법 중 편한 것을 고르세요.

#### 방법 1. 앱 강의 화면의 Colab 버튼으로 바로 열기 (가장 빠름)

각 레슨 강의 화면 상단의 `Open in Colab` 버튼을 클릭합니다.

- 학생용 버튼: `[학생용] 레슨 NN — 제목.ipynb`
- 선생님용 버튼: `레슨 NN — 제목.ipynb` (교사/관리자에게만 표시)

#### 방법 2. GitHub 링크로 바로 열기

Colab에서 `파일 → GitHub에서 노트북 열기`를 선택하고 저장소와 파일 경로를 입력합니다.

#### 방법 3. 파일 → 노트북 업로드

레슨 폴더에서 `.ipynb` 파일을 PC에 다운로드한 뒤,
코랩 메뉴 `파일 → 노트북 업로드` 에서 선택합니다.

### 3. 데이터 파일 가져오기

레슨마다 `data/` 폴더에 HTML, CSV, TXT fixture가 있습니다. 코랩에서는 기본적으로 노트북 첫 셀이 GitHub raw URL에서 파일을 읽습니다.

**옵션 A. 노트북의 `load_text()` 그대로 사용**

```python
html = load_text("mini_shop.html")
print(len(html))
```

**옵션 B. 직접 업로드**

코랩 좌측 폴더 아이콘 → 업로드. raw GitHub URL 배포 전 검수에 적합합니다.

**옵션 C. 구글 드라이브 마운트**

```python
from google.colab import drive
drive.mount("/content/drive")
```

데이터를 본인 드라이브에 미리 올려둔 경우 사용합니다.

### 4. 라이브러리 설치

코랩은 `requests` 가 기본 설치되어 있는 경우가 많지만, 환경에 따라 `beautifulsoup4` 가 없을 수 있습니다. 모든 레슨 첫 셀은 필요한 라이브러리가 없으면 자동 설치합니다.

```python
try:
    import requests
    from bs4 import BeautifulSoup
except ImportError:
    import sys, subprocess
    subprocess.check_call([sys.executable, "-m", "pip", "-q", "install", "requests", "beautifulsoup4"])
```

브라우저 자동화 레슨(6강 이후)에서는 Playwright 또는 Selenium 설치 셀이 별도로 제공됩니다.

### 5. 결과 저장 / 제출

`파일 → .ipynb 다운로드` 로 노트북을 PC에 받고, 생성된 CSV/JSON 파일과 함께 우리 사이트의 과제 제출 화면 또는 강사가 지정한 곳에 업로드합니다.

## B. VSCode + Jupyter 로 진행하는 방법

코딩 환경이 익숙한 학생은 이쪽이 더 편합니다.

### 1. 사전 설치

```bash
# Python 3.10 이상 권장
pip install jupyter jupyterlab requests beautifulsoup4 pandas
```

브라우저 자동화 레슨에서는 강사 안내에 따라 다음 중 하나를 추가 설치합니다.

```bash
pip install playwright
python -m playwright install chromium
```

또는

```bash
pip install selenium
```

### 2. 확장 설치

VSCode 에서 다음 두 확장을 설치합니다.

- **Python** (Microsoft)
- **Jupyter** (Microsoft)

### 3. 노트북 열기

레슨 폴더의 `.ipynb` 를 VSCode 에서 그대로 엽니다. 첫 실행 시 우상단에서 파이썬 인터프리터를 선택하라는 안내가 뜹니다.

### 4. 데이터 경로

VSCode 로컬 실행이면 노트북 첫 셀의 `DATA_BASE` 가 `"./data"` 로 잡힙니다. 별도 다운로드 없이 그대로 진행 가능.

### 5. 결과 저장

`.ipynb` 파일과 생성된 CSV/JSON 파일을 그대로 강사 지정 위치에 업로드합니다.

## C. 노트북 첫 셀 — 환경 감지 코드

모든 레슨 노트북 첫 코드 셀은 다음과 같이 시작합니다. **수정하지 마세요.**

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

이 셀은 학생이 코랩이든 VSCode든 같은 코드로 fixture 데이터를 읽을 수 있게 해 줍니다.

## D. 앱 Colab URL 규칙

앱의 강의 화면은 코스의 `notebookBaseUrl`과 레슨 제목으로 코랩 URL을 자동 생성합니다.

- 권장 `notebookBaseUrl`: `https://colab.research.google.com/github/Kevin-innovation/jupyter-lecture/blob/main/python-web-automation`
- 관리자 가져오기 화면에서 이 코스를 스캔 후 저장하면 위 URL이 코스 설정에 자동 반영됩니다.
- 학생용: `{notebookBaseUrl}/lectures/NN/[학생용] 레슨 NN — 제목.ipynb`
- 선생님용: `{notebookBaseUrl}/lectures/NN/레슨 NN — 제목.ipynb`

따라서 각 레슨 폴더에는 원본 분리 파일과 별도로 앱이 여는 통합 노트북 2개를 반드시 둡니다.

- `[학생용] 레슨 NN — 제목.ipynb`: 강의 + 학생용 문제 + 최종 미션, 정답 없음
- `레슨 NN — 제목.ipynb`: 강의 + 문제별 정답/해설 + 최종 미션 모범 답안 + 교사용 가이드

## E. 안전한 웹 자동화 기준

이 코스에서는 다음 기준을 기본으로 둡니다.

- `requests.get(..., timeout=10)` 처럼 timeout을 둡니다.
- 응답을 받은 뒤 `status_code` 또는 `raise_for_status()` 로 실패를 확인합니다.
- 외부 사이트 요청 시 User-Agent를 명시하고, 요청 간격을 둡니다.
- 실제 사이트를 반복 요청하기 전에 약관과 robots.txt를 확인합니다.
- 로그인 우회, 유료 데이터 무단 수집, 개인정보 수집, 공격성 요청은 하지 않습니다.
- 수업 중 대량 요청이 필요해 보이면 fixture 파일로 바꿉니다.

## F. 자주 묻는 질문

**Q. 코랩에서 실행한 결과를 다시 열면 사라져요.**
A. `.ipynb` 자체는 본인 드라이브에 저장되지만 임시 파일(다운받은 HTML/CSV 등)은 사라집니다. 매번 첫 환경 셀을 다시 실행하면 됩니다.

**Q. 코랩 무료 버전으로 충분한가요?**
A. 1~5강의 HTTP/HTML 파싱은 무료 버전으로 충분합니다. 브라우저 자동화도 수업용 작은 예제는 무료 버전으로 충분합니다.

**Q. `403`, `429` 같은 에러가 떠요.**
A. 외부 사이트가 요청을 거부하거나 너무 많은 요청을 받았다는 뜻일 수 있습니다. 수업 중에는 즉시 요청을 멈추고 fixture 데이터로 돌아갑니다.

**Q. selector 결과가 비어 있어요.**
A. `len(soup.select("..."))` 로 먼저 개수를 확인합니다. class 앞의 `.`, id 앞의 `#`, 태그 이름 오타를 확인하세요.

**Q. 노트북을 강사에게 어떻게 보여주나요?**
A. `.ipynb` 그대로 제출하거나, 생성된 CSV/JSON 파일과 함께 제출합니다. 코드와 출력이 함께 보여야 합니다.

**Q. 실제 사이트를 직접 수집해도 되나요?**
A. 강사가 지정한 범위 안에서만 가능합니다. 약관, robots.txt, 요청 간격, 개인정보 여부를 확인하지 않은 사이트는 사용하지 않습니다.

## G. 환경 점검 셀

수업 시작 전에 한 번 이 셀을 실행해서 환경이 정상인지 확인하세요.

```python
import sys
import requests
from bs4 import BeautifulSoup

print("python   :", sys.version.split()[0])
print("requests :", requests.__version__)
print("bs4      :", BeautifulSoup("<p>ok</p>", "html.parser").p.text)
```

정상이면 Python 버전, requests 버전, `ok` 가 출력됩니다. 오류가 나면 첫 환경 셀을 다시 실행합니다.
