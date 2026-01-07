# 🔥 Team Git & Collaboration Convention

## 1️⃣ 코드 컨벤션

### Ruff (Python Linter & Formatter)

Ruff는 Rust로 작성된 고성능 Python 린터이자 코드 포매터다. 기존의 Python 코드 품질 도구들(Black, flake8, isort 등)을 하나로 통합하면서도 극도로 빠른 성능을 제공한다.

#### 설치 및 설정

```bash
# uv 설치
pip install uv

# 프로젝트 초기화 (pyproject.toml 생성)
uv init

# 패키지 설치
uv add [ex. requests]

# Ruff 개발 의존성으로 설치
uv add --group dev ruff
```

#### VSCode 설정

`pyproject.toml`에서 Ruff 설정 후, VSCode의 lint 설정을 Ruff로 변경하면 자동으로 적용된다.

프로젝트 루트에 `.vscode/settings.json` 파일을 커밋하여 모든 팀원의 개발 환경을 통일할 수 있다.

> **참고**: `uv init`은 필수가 아니다. 기존 프로젝트 구조가 있다면 `pyproject.toml` 파일만 있으면 `uv`를 사용할 수 있다.

---

### Docstring 컨벤션 (Google Style)

모든 함수는 Google Style Docstring을 사용한다.

```python
def function(arg1: type, arg2: type) -> return_type:
    """함수 요약 설명 (한 줄)

    필요한 경우 추가 설명을 한 줄 띄우고 작성한다.
    여러 줄 설명도 가능하다.

    Args:
        arg1 (type): 인자 설명
        arg2 (type): 인자 설명

    Returns:
        return_type: 반환 값 설명

    Raises:
        ErrorType: 발생 가능한 예외에 대한 설명
    """
    pass
```

---

## 2️⃣ 브랜치 전략 — GitHub Flow

> **원칙**: `main` 브랜치는 항상 배포 가능한 상태를 유지한다.

| Branch | 사용 목적 |
| --- | --- |
| `main` | 안정 배포 브랜치 |
| `dev/<issue-number>-<issue-name>` | 새로운 기능 개발 |

### 브랜치 명명 규칙

브랜치 이름은 이슈 번호와 이슈 이름을 조합하여 작성한다.

**예시:**
```
feature/12-ingest-eco-price
hotfix/21-streamlit-refresh-error
```

---

## 3️⃣ 이슈 관리

### 이슈 작성 규칙

- 이슈 이름은 **영문 기반으로 짧고 명확하게** 작성한다
- 필요한 경우 본문 또는 라벨에서 한글 설명 추가
- 브랜치 이름에 이슈 번호와 제목을 그대로 사용

### 이슈 제목 예시

```
Ingest-eco-data : 한글 설명
add gold-layer : Add gold table for analytics
streamlit refresh error : Fix streamlit refresh error in website
```

### 📝 Issue Template

```markdown
## ✨ Summary
- 무엇을 할까요?

## 🎯 Goal
- 반영되는 파이프라인 단계: Raw / Silver / Gold

## 📌 Tasks
- [ ] TODO 1
- [ ] TODO 2
```

---

## 4️⃣ PR (Pull Request) 규칙

### PR 작성 규칙

- 해당 이슈 번호를 반드시 포함
- 리뷰어 지정 & Label 필수
- 머지 전 **CI 통과 + Self Review 체크**

### PR 제목 형식

```
<브랜치 이름>(#<이슈번호>)
```

**예시:**
```
Ingest-eco-data(#12)
add gold-layer(#18)
```

### 📝 PR Template

```markdown
## ✨ What
- 무슨 작업을 했는지

## 🎯 Why
- 왜 필요한 작업인지  
- 링크: Closes #12

## 📌 Changes
- 주요 변경 사항 bullet 정리

## 🧪 Test
- 테스트 방법 또는 로컬 실행 결과

## 🤔 Request for Review
- 리뷰어에게 특히 피드백을 받고 싶은 부분
- 구조적으로 맞는지 확신이 없는 부분
- 다른 대안이 있다고 생각되는 부분
- 성능/구현 방식에서 고민한 지점
```

---

## 5️⃣ 커밋 컨벤션 (Conventional Commits)

### 커밋 메시지 형식

```
<type>: <message> (#<issue-number>)
```

### 커밋 예시

```
feat: add eco price ingestion pipeline (#12)
fix: wrong query on gold table (#18)
docs: update data schema design (#05)
```

### 🔖 타입 목록

| type | 설명 |
| --- | --- |
| `feat` | 새로운 기능 추가 (수집/ETL/처리 기능 등) |
| `fix` | 버그 수정 |
| `data` | 스키마/정제/테스트 데이터 수정 |
| `refactor` | 기능 변화 없이 코드 개선 |
| `docs` | 문서 작업 (`README`, `docs/` 등) |
| `config` | 환경 설정 변경 (Airflow, Docker, CI/CD 등) |
| `test` | 테스트 코드 관련 |
| `chore` | 기타 자잘한 작업 |
