---
name: Feature Task
about: SRS 기반의 구체적인 개발 태스크 명세
title: "[DB] DB-011: 멀티테넌트 Row-Level Isolation 아키텍처 연동"
labels: 'database, security, priority:critical, phase:1'
assignees: ''
type: task
tags: [task, database]
created: 2026-04-22
---
## 🎯 Summary
- 기능명: [DB-011] SaaS 멀티테넌트 핵심 — Row-Level Isolation 연동 체계
- 목적: B2B SaaS 앱의 핵심인 REQ-FUNC-026 (Row-Level Security)을 코드 레벨에 구현. 한 테넌트의 유저가 다른 테넌트의 DB(주문, 재고, 예측치)에 질의(Query)할 수 없도록, SQLAlchemy 이벤트 훅이나 전역 Base Repository 단에 자동 `where(tenant_id=...)` 필터를 적용하는 레이어를 구성한다.

## 🔗 References (Spec & Context)
- SRS 멀티테넌트: [`SRS-V1.0.md#§4.1.4`](file:///e:/workspace/SRS-from-PRD/SRS-V1.0.md) — REQ-FUNC-026 (tenant_id 기준 격리 강제)
- SRS 보안: [`SRS-V1.0.md#§4.2.3`](file:///e:/workspace/SRS-from-PRD/SRS-V1.0.md) — REQ-NF-018 (화주사 간 데이터 격리 테스트 가능성)

## ✅ Task Breakdown (실행 계획)

### 1. 전역 Base Repository 추상화 클래스 생성
- [ ] `app/adapters/persistence/base.py` 로 제네릭 CRUD 뼈대 작성:
  ```python
  from sqlmodel.ext.asyncio.session import AsyncSession
  from sqlmodel import select
  from typing import TypeVar, Generic, Type, Optional, List
  import uuid

  ModelType = TypeVar("ModelType", bound=SQLModel)

  class BaseTenantRepository(Generic[ModelType]):
      def __init__(self, model: Type[ModelType]):
          self.model = model

      # 격리 강제 조회
      async def get(self, session: AsyncSession, tenant_id: uuid.UUID, id: Any) -> Optional[ModelType]:
          query = select(self.model).where(
              self.model.id == id,
              self.model.tenant_id == tenant_id  # <--- 격리의 핵심
          )
          result = await session.execute(query)
          return result.scalar_one_or_none()
          
      # 격리 강제 다건 조회
      async def get_multi(self, session: AsyncSession, tenant_id: uuid.UUID, skip: int = 0, limit: int = 100) -> List[ModelType]:
          query = select(self.model).where(self.model.tenant_id == tenant_id).offset(skip).limit(limit)
          result = await session.execute(query)
          return result.scalars().all()
  ```

### 2. 하위 도메인 Repository 전면 리팩토링
- [ ] DB-002, DB-003, DB-004, DB-007, DB-008, DB-009 에서 정의한 `tenant_id` 종속 엔터티들의 레포지토리가 이 `BaseTenantRepository` 를 상속받아 일관되게 검증 파라미터(`tenant_id: UUID`)를 요구하도록 강제 변경.
  *(예외: WEATHER_DATA 등 글로벌 공용 데이터는 상속 금지)*

### 3. FastAPI 의존성 조합 방안 (Service 층)
- [ ] `tenant_id` 를 의존성(`get_current_user` 등)에서 뽑아 리파지토리에 전달하는 체인 문서화 및 구현.

## 🧪 Acceptance Criteria (BDD/GWT)

**Scenario 1: 교차 테넌트 접근 원천봉쇄**
- Given: A 테넌트의 사용자 세션(`tenant_id="A"`)이 발급됨
- When: URL 조작 등 해킹으로 B 테넌트 소유의 오더 ID(`order_id="B_123"`)를 조회 요청함
- Then: `BaseTenantRepository` 의 `where` 절 불일치로 무조건 `None` 이 반환되어 404 Not Found 로 귀결되고 데이터 유출이 방어됨.

**Scenario 2: 글로벌 공용 테이블 접근 (예외)**
- Given: 공용 날씨 테이블(`WEATHER_DATA`) 의 인출 상황
- When: A 테넌트 유저가 해당 데이터를 요청함
- Then: 이 모델은 테넌트 베이스 클래스(`BaseTenantRepository`)를 상속하지 않았으므로, 격리 조건 쿼리가 적용되지 않고 정상적으로 공용 날씨 데이터가 검색된다.

## ⚙️ Technical Constraints
- PostgreSQL 자체 RLS 정책(`CREATE POLICY`)을 사용하는 방법도 있으나, 애플리케이션 레벨(ORM)에서 제어하는 것이 AI Agent 기반 Python 스택 코드(SQLModel) 유지보수 및 디버깅에 유리하므로 어플리케이션 계층 필터(Repository Layer)를 권장한다.

## 🏁 Definition of Done (DoD)
- [ ] `BaseTenantRepository` 제네릭 클래스 작성이 완료되었나?
- [ ] tenant를 가진 모든 하위 엔터티 저장/조회 쿼리 로직에 `tenant_id` 필터링이 씌워졌는가? (코드 리뷰 확인 방안 포함)
- [ ] pytest 기반 타 테넌트 고립도 방어 유닛 테스트 코드가 작성되고 통과했는가?
