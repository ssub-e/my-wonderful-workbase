---
name: Feature Task
about: SRS 기반의 구체적인 개발 태스크 명세
title: "[Infra] ENV-004: Python 의존성 관리 및 린터/포매터 설정"
labels: 'infra, tools, priority:high, phase:0'
assignees: ''
type: task
tags: [task, infrastructure]
created: 2026-04-22
---
## 🎯 Summary
- 기능명: [ENV-004] Python 패키지 의존성 관리(`pyproject.toml`) 및 Linter/Formatter (`Ruff`) 셋업
- 목적: 전체 프로젝트에서 사용할 모든 주요 라이브러리(FastAPI, SQLModel, LightGBM, SHAP 등)의 버전을 확립하고 중앙 관리한다. 또한 코드 협업 품질을 위한 정적 코드 분석기(Ruff) 룰을 세팅하여 1개의 커밋이라도 스타일을 어기면 CI를 거절하도록 통제한다.

## 🔗 References (Spec & Context)
- SRS 기술 스택: [`SRS-V1.0.md#§1.5.3`](raw/assets/SRS-V1.0.md) — C-TEC 전체 모듈 목록 참고
- SRS 모듈 제약: C-TEC-001(Streamlit), 002(FastAPI), 003(SQLModel/asyncpg), 004(redis.asyncio), 005(lightgbm), 006(shap), 007(weasyprint/fpdf2), 008(apscheduler)

## ✅ Task Breakdown (실행 계획)

### 1. `pyproject.toml` 작성 및 패키지 버저닝
- [ ] 프로젝트 루트에 `pyproject.toml` 설계. `uv` 또는 `poetry` 문법 기반:
  ```toml
  [project]
  name = "ai-demand-forecast-saas"
  version = "1.0.0"
  requires-python = ">=3.11"
  dependencies = [
      "fastapi[all]>=0.110.0",
      "sqlmodel>=0.0.16",
      "alembic>=1.13.1",
      "asyncpg>=0.29.0",
      "redis>=5.0.3",
      "apscheduler>=3.10.4",
      "lightgbm>=4.3.0",
      "shap>=0.45.0",
      "pydantic-settings>=2.2.1",
      "passlib[bcrypt]>=1.7.4",
      "python-jose[cryptography]>=3.3.0",
      "weasyprint>=61.0",
      "fpdf2>=2.7.8", # (백업용)
      "streamlit>=1.33.0",
      "recharts" # (streamlit component용)
  ]
  ```

### 2. Ruff 린터 및 포매터 룰 세팅
- [ ] `pyproject.toml` 내부 도구 셋업 (Ruff는 기존 Pylint/Flake8/Black/Isort를 하나로 대체):
  ```toml
  [tool.ruff]
  line-length = 100
  target-version = "py311"

  [tool.ruff.lint]
  select = [
      "E",  # pycodestyle errors
      "W",  # pycodestyle warnings
      "F",  # pyflakes
      "I",  # isort
      "C",  # flake8-comprehensions
      "B",  # flake8-bugbear
  ]
  ignore = [
      "E501",  # line too long, handled by black
      "B008",  # do not perform function calls in argument defaults
  ]
  ```

### 3. 개발/테스트 격리 그룹 설정
- [ ] `[project.optional-dependencies]` 구성:
  ```toml
  dev = [
      "pytest>=8.1.1",
      "pytest-asyncio>=0.23.6",
      "pytest-cov>=5.0.0",
      "httpx>=0.27.0",
      "locust>=2.24.0" # 부하테스트용
  ]
  ```

## 🧪 Acceptance Criteria (BDD/GWT)

**Scenario 1: 일관된 포맷팅 강제**
- Given: 들여쓰기와 import 순서가 엉망인 `main.py`
- When: 개발자가 터미널에서 `ruff format .` 과 `ruff check . --fix` 를 실행함
- Then: 파일 내의 import 구문이 사전순으로 정렬(isort)되고, 구문이 단일 컨벤션으로 깔끔하게 포매팅되어 저장된다.

**Scenario 2: 의존성 설치 호환성 검증**
- Given: 비어있는 Python 3.11 가상환경(`venv`)
- When: `pip install -e .[dev]` 명령으로 인스톨함
- Then: C-TEC 스택에 나열된 모든 필수 머신러닝 라이브러리(LightGBM, Shap)와 백엔드 라이브러리 간 의존성 충돌 없이 인스톨이 완료된다.

## ⚙️ Technical Constraints
- `Python` >= 3.11 버전을 강제하여 최신 타이핑 문법 및 에러메시지 개선 이점을 누린다.
- 추후 Docker 빌드시 `pyproject.toml`을 활용한 빌드 캐싱 구조를 구성해야 한다.

## 🏁 Definition of Done (DoD)
- [ ] `pyproject.toml` 에 지정된 SRS 기술 스택이 모두 기재되었는가?
- [ ] `ruff` 린터와 포매터 룰 블럭이 포함되었는가?
- [ ] 터미널에서 전체 코드 스캔 결과 0 Erros 를 확인했는가?
