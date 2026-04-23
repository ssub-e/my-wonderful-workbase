---
name: Feature Task
about: SRS 기반의 구체적인 개발 태스크 명세
title: "[Test] TEST-001: 백엔드 단위 테스트(Pytest) 파이프라인 및 DB Mocking 구조"
labels: 'test, qa, backend, phase:7, priority:critical'
assignees: ''
type: task
tags: [task, testing]
created: 2026-04-22
---
## 🎯 Summary
- 기능명: [TEST-001] FastAPI 코어 로직 검증용 Pytest 프레임워크 셋업
- 목적: B2B 시스템의 안전성을 담보하기 위해, GitHub Actions(`ENV-003`)에서 매 푸시마다 자동으로 백엔드 코드를 검증할 테스트 프레임워크를 수립한다. 실제 DB를 건드리지 않고 인메모리(SQLite) 혹은 Rollback 세션을 사용한다.

## 🔗 References (Spec & Context)
- 기술 스택: `pytest`, `pytest-asyncio`, `httpx`

## ✅ Task Breakdown (실행 계획)

### 1. 테스트 의존성 및 `conftest.py` 설정
- [ ] `requirements-dev.txt`에 `pytest-asyncio`, `pytest-cov`, `httpx` 추가.
- [ ] `tests/conftest.py` 작성:
  ```python
  import pytest
  from fastapi.testclient import TestClient
  from sqlmodel import Session, SQLModel, create_engine
  from sqlmodel.pool import StaticPool
  from app.main import app
  from app.core.database import get_session

  # 인메모리 SQLite 엔진 (테스트 속도 극대화)
  engine = create_engine(
      "sqlite://", 
      connect_args={"check_same_thread": False}, 
      poolclass=StaticPool
  )

  @pytest.fixture(name="session")
  def session_fixture():
      SQLModel.metadata.create_all(engine)
      with Session(engine) as session:
          yield session
      SQLModel.metadata.drop_all(engine)

  @pytest.fixture(name="client")
  def client_fixture(session: Session):
      def get_session_override():
          return session
      # 프로덕션 DB 통신 함수를 테스트용 인메모리 함수로 덮어치기(Dependency Override)
      app.dependency_overrides[get_session] = get_session_override
      client = TestClient(app)
      yield client
      app.dependency_overrides.clear()
  ```

### 2. 기본 API 엔드포인트 테스트 스크립트 작성
- [ ] `tests/test_health.py` 작성하여 `/health` 체크 무결성 및 응답 속도 확인 검사.

## 🧪 Acceptance 초도 기준 (BDD/GWT)

**Scenario 1: API 통신간 세션 데이터 롤백 (독립성 보장)**
- Given: 테스트 1번(`test_create_user`)이 먼저 인메모리 DB에 '김철수' 데이터를 INSERT 함.
- When: 바로 이어서 테스트 2번(`test_read_users`)이 전체 유저를 조회함.
- Then: `conftest.py` 의 `session_fixture` 제어 로직에 의해 직전 테스트가 끝날때마다 트랜잭션이 초기화(`drop_all`) 되므로, 테스트 2번은 깨끗한 0개의 레코드 상태에서 시작하여 **테스트 간의 상호 간섭(Side-effect)**을 완벽히 차단한다.

## ⚙️ Technical Constraints
- 비동기(Async) 라우터를 테스트하려면 `pytest-asyncio` 마커(`@pytest.mark.asyncio`)와 `httpx.AsyncClient` 를 상황에 맞게 섞어 써야 함을 유의한다.

## 🏁 Definition of Done (DoD)
- [ ] Dependency Overrides 기법을 이용한 DB 세션 프록시 픽스처가 존재하는가?
- [ ] `pytest --cov` 명령어를 통해 커버리지 레포트가 터미널에 출력되는가?
