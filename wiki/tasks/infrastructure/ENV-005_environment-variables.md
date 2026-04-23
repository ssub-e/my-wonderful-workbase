---
name: Feature Task
about: SRS 기반의 구체적인 개발 태스크 명세
title: "[Infra] ENV-005: 환경 변수 관리 체계 (Pydantic Settings)"
labels: 'infra, security, priority:high, phase:0'
assignees: ''
type: task
tags: [task, infrastructure]
created: 2026-04-22
---
## 🎯 Summary
- 기능명: [ENV-005] 환경 변수 관리 체계 (.env / Secrets) 구축
- 목적: 데이터베이스, Redis, 타겟 API, 보안 난수 키(SECRET_KEY), Gemini API Key 등 프로젝트 전반에 쓰이는 민감 정보를 코드에서 분리하고, `pydantic-settings` 모듈을 통해 타입 힌트와 필수값 유무 검증을 강제한다 (C-TEC-009 연계).

## 🔗 References (Spec & Context)
- SRS 외부 시스템 연동: [`SRS-V1.0.md#§3.1`](raw/assets/SRS-V1.0.md) — EXT-01~07 (각종 외부 API 연동 토큰 관리 필요)
- SRS 보안 인프라: C-TEC-009 (Gemini 환경변수 전환) 및 시크릿 하드코딩 금지 룰.

## ✅ Task Breakdown (실행 계획)

### 1. Pydantic Settings 클래스 구현
- [ ] `app/core/config.py` 작성:
  ```python
  from pydantic_settings import BaseSettings, SettingsConfigDict
  from typing import Optional

  class Settings(BaseSettings):
      PROJECT_NAME: str = "B2B AI Demand Forecasting SaaS"
      VERSION: str = "1.0.0"
      API_V1_STR: str = "/api/v1"
      
      # Database & Cache
      DATABASE_URL: str
      REDIS_URL: str
      
      # Security
      SECRET_KEY: str  # JWT 서명용 대칭키 (openssl rand -hex 32)
      ACCESS_TOKEN_EXPIRE_MINUTES: int = 30
      
      # External API Keys & Webhooks
      GEMINI_API_KEY: str
      SLACK_WEBHOOK_URL: Optional[str] = None
      CAFE24_CLIENT_ID: Optional[str] = None
      CAFE24_CLIENT_SECRET: Optional[str] = None
      
      # Environment Flag
      ENVIRONMENT: str = "development" # development, staging, production
      ECHO_SQL: bool = False

      model_config = SettingsConfigDict(env_file=".env", case_sensitive=True)

  settings = Settings()
  ```

### 2. .env 템플릿 배포
- [ ] 루트 패스에 `.env.example` 작성. 팀원 공유 용도이므로 절대 실 서버 패스워드를 적지 않음.
  ```env
  DATABASE_URL=postgresql://user:pass@localhost:5432/saas_db
  REDIS_URL=redis://localhost:6379/0
  SECRET_KEY=여기에_랜덤_해시를_넣으세요
  GEMINI_API_KEY=your_gemini_api_key_here
  ```
- [ ] `.gitignore` 에 `.env` 추가되었는지 최종 점검.

## 🧪 Acceptance Criteria (BDD/GWT)

**Scenario 1: 필수 환경 변수 누락 방어**
- Given: `.env` 파일에 `GEMINI_API_KEY` 환경변수가 주석처리 되어 있음
- When: 애플리케이션 (FastAPI) 기동 (`from app.core.config import settings` 로드 시점)
- Then: `pydantic.ValidationError` 구문 분석 에러와 동시에 `GEMINI_API_KEY` 필드 필수 에러(required)가 로깅되며, 앱 기동이 즉각 중단된다.

**Scenario 2: 데이터 타입 형변환(Casting) 확인**
- Given: `.env` 환경 변수로 문자열 `ACCESS_TOKEN_EXPIRE_MINUTES="60"` 이 들어옴 
- When: 로직에서 `settings.ACCESS_TOKEN_EXPIRE_MINUTES` 을 호출함
- Then: 이 값은 런타임에 파이썬 원시 `int` 형(정수 60)으로 변환되어 안전하게 계산 로직에 투입 가능하다.

## ⚙️ Technical Constraints
- `pydantic-settings` 라이브러리를 활용할 것. V2 버전 문법(`SettingsConfigDict`)을 준수.
- 테스트 환경(`pytest`) 작동 시에는 로컬 `.env` 에 독립될 수 있도록, `conftest.py` 등에서 몽키패칭하여 Dummy 키들을 주입하는 가이드라인 확보 고려.

## 🏁 Definition of Done (DoD)
- [ ] `core/config.py` 내에 Settings 객체 인스턴스화 코드가 완성되었는가?
- [ ] `DATABASE_URL`, `SECRET_KEY`, `GEMINI_API_KEY` 필수 파라미터가 검증되는가?
- [ ] 실제 개발 서버 구동 시 에러 없이 로드되는가?
- [ ] `.gitignore` 적용 점검 확인?
