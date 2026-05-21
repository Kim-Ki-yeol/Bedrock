# 코드 리뷰 리포트

## student_system.py (287줄)

### 스타일 검사

PEP8 및 일반적인 스타일 규칙 위반 목록:

**명명 규칙 (Naming Convention)**
- [12] 보안/스타일 위반: `admin_password`에 하드코딩된 비밀번호 ("kimkiyeol123") — 자격증명은 코드에 직접 포함하면 안 됨
- [134] 스타일 위반: `report_title`에 하드코딩된 고유 이름 포함 — 매직 스트링은 상수(UPPER_SNAKE_CASE)로 분리 권장

**코드 스타일**
- [57] 스타일 위반: `total = total + scores[subject]` → `total += scores[subject]`로 작성하는 것이 Pythonic함
- [55] 스타일 위반: `if len(scores) == 0:` → `if not scores:`로 작성하는 것이 Pythonic함
- [39] 스타일 위반: `for student_id in self.students: result.append(self.students[student_id])` → `return list(self.students.values())`로 간결하게 표현 가능
- [170~174] 스타일 위반: 문자열 연결(`+`)을 반복 사용 — f-string 또는 `str.format()` 사용 권장 (여러 곳에서 반복됨)

**보안 위반**
- [123] 보안 위반: `execute_search_query` 메서드에서 SQL 쿼리를 문자열 연결로 조합 — SQL Injection 취약점 (OWASP A03: Injection)
- [246] 보안 위반: `unsafe_html_report`에서 사용자 입력을 HTML에 직접 삽입 — XSS 취약점 (OWASP A03: Injection)
- [250] 보안 위반: `debug_print_secret` 메서드에 하드코딩된 시크릿 키 노출 및 디버그 출력 — 프로덕션 코드에 존재하면 안 됨

**함수/메서드 설계**
- [118] 스타일 위반: `login_admin` 메서드에서 `if password == ...: return True \n return False` → `return password == self.admin_password`로 단순화 가능
- [250] 스타일 위반: `debug_print_secret`은 디버그용 메서드로 실제 코드에 포함되어선 안 됨

**독스트링 누락**
- [9, 127, 263, 271] 스타일 위반: 클래스 및 주요 함수에 docstring이 없음 — PEP8은 공개 모듈, 함수, 클래스, 메서드에 docstring 작성을 권장

---

### 보안 검사

## 🔴 취약점 1: 하드코딩된 비밀번호 (심각도: 높음)

**위치:** `StudentManager.__init__()` 및 `ReportGenerator.debug_print_secret()`

```python
self.admin_password = "kimkiyeol123"  # StudentManager
secret_key = "debug-secret-key-kimkiyeol"  # ReportGenerator
```

**문제점:**
- 소스코드에 비밀번호/시크릿 키가 평문으로 하드코딩되어 있음
- 코드 저장소(GitHub 등)에 노출될 경우 즉시 탈취 가능
- `debug_print_secret()`이 운영 환경에서도 시크릿 키를 출력할 수 있음

**수정 제안:**

```python
import os
import hashlib

# 환경변수에서 읽거나, 해시값으로 비교
self.admin_password_hash = os.environ.get("ADMIN_PASSWORD_HASH")

def login_admin(self, password):
    input_hash = hashlib.sha256(password.encode()).hexdigest()
    return input_hash == self.admin_password_hash
```

---

## 🔴 취약점 2: SQL Injection (심각도: 높음)

**위치:** `StudentManager.execute_search_query()`

```python
def execute_search_query(self, keyword):
    query = "SELECT * FROM students WHERE name = '" + keyword + "'"
    return query
```

**문제점:**
- 사용자 입력값을 검증 없이 SQL 쿼리에 직접 연결(concatenation)
- 공격자가 `keyword`에 `' OR '1'='1` 같은 값을 입력하면 모든 데이터 탈취 가능
- `'; DROP TABLE students; --` 입력 시 테이블 삭제 가능

**수정 제안:**

```python
# Parameterized Query (prepared statement) 사용
def execute_search_query(self, cursor, keyword):
    query = "SELECT * FROM students WHERE name = ?"
    cursor.execute(query, (keyword,))
    return cursor.fetchall()
```

---

## 🔴 취약점 3: XSS (Cross-Site Scripting) (심각도: 높음)

**위치:** `ReportGenerator.unsafe_html_report()`

```python
def unsafe_html_report(self, title, content):
    html = "<h1>" + title + "</h1>"
    html = html + "<div>" + content + "</div>"
    return html
```

**문제점:**
- `title`이나 `content`에 `<script>alert('XSS')</script>` 같은 값이 입력되면 그대로 HTML에 삽입됨
- 해당 HTML이 브라우저에서 렌더링될 경우 악성 스크립트 실행 가능
- 세션 탈취, 피싱 등 2차 공격으로 이어질 수 있음

**수정 제안:**

```python
import html

def unsafe_html_report(self, title, content):
    safe_title = html.escape(title)
    safe_content = html.escape(content)
    result = "<h1>" + safe_title + "</h1>"
    result = result + "<div>" + safe_content + "</div>"
    return result
```

또는 `markupsafe`, `bleach` 같은 라이브러리 활용 권장

---

## 🟡 취약점 4: 입력값 검증 미흡 (심각도: 중간)

**위치:** `main()` 함수 내 `choice == "3"` 처리 부분

```python
score = int(input("점수: "))  # 숫자가 아닌 값 입력 시 예외 발생
```

**문제점:**
- 사용자가 숫자가 아닌 값(예: `abc`) 입력 시 `ValueError` 발생하여 프로그램이 비정상 종료됨
- 입력값에 대한 예외 처리가 전혀 없음

**수정 제안:**

```python
try:
    score = int(input("점수: "))
except ValueError:
    print("올바른 숫자를 입력하세요.")
    continue
```

---

## 📋 취약점 요약

| # | 취약점 유형 | 위치 | 심각도 |
|---|------------|------|--------|
| 1 | 하드코딩된 비밀번호 | `__init__`, `debug_print_secret` | 🔴 높음 |
| 2 | SQL Injection | `execute_search_query` | 🔴 높음 |
| 3 | XSS | `unsafe_html_report` | 🔴 높음 |
| 4 | 입력값 검증 미흡 | `main()` | 🟡 중간 |

> 특히 1~3번 취약점은 OWASP Top 10에 포함된 대표적인 보안 취약점으로, 즉시 수정이 필요합니다.

---