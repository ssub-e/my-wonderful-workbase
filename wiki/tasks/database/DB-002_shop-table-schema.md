---
name: Feature Task
about: SRS 기반의 구체적인 개발 태스크 명세
title: "[DB] DB-002: SHOP 테이블 스키마 + SQLModel 모델 + Repository"
labels: 'database, backend, priority:high, phase:1'
assignees: ''
type: task
tags: [task, database]
created: 2026-04-22
---
## 🎯 Summary
- 기능명: [DB-002] SHOP 테이블 스키마, SQLModel 정의, DTO 및 레포지토리
- 목적: 다중 화주사(Tenant) 환경하에서, 화주사들이 자신의 상점 단위 연동 플랫폼(카페24, 스마트스토어 등)의 OAuth 토큰 및 연동 진행 상태를 담고 갱신할 물리적 엔터티 공간을 마련한다. 

## 🔗 References (Spec & Context)
- SRS ERD: [`SRS-V1.0.md#§6.2`](raw/assets/SRS-V1.0.md) — TENANT 하위 엔터티
- 연동 제약 기능: [`SRS-V1.0.md#§4.1.2`](raw/assets/SRS-V1.0.md) — REQ-FUNC-016 (연동 상태 status: connected/disconnected 기록)

## ✅ Task Breakdown (실행 계획)

### 1. SQLModel 스키마 정의
- [ ] `app/domain/integration/models.py` 내의 `Shop` 정의:
  ```python
  from sqlmodel import SQLModel, Field
  import uuid
  from datetime import datetime
  from typing import Optional

  class Shop(SQLModel, table=True):
      __tablename__ = "shops"
      
      id: uuid.UUID = Field(default_factory=uuid.uuid4, primary_key=True)
      tenant_id: uuid.UUID = Field(foreign_key="tenants.id", index=True, nullable=False)
      platform: str = Field(max_length=50, description="cafe24, smartstore 등")
      oauth_token: Optional[str] = Field(default=None) # TODO: 민감필드
      refresh_token: Optional[str] = Field(default=None) 
      status: str = Field(default="disconnected", description="connected / disconnected / error")
      last_sync: Optional[datetime] = Field(default=None)
      created_at: datetime = Field(default_factory=datetime.utcnow)
  ```

### 2. 보안 DTO 구성 (API 응답용)
- [ ] `app/schemas/integration_schema.py`:
  ```python
  class ShopRead(BaseModel):
      id: uuid.UUID
      platform: str
      status: str
      last_sync: Optional[datetime]
      # oauth_token 등은 응답에 포함시키지 않음 (보안)
  ```

### 3. DB Repository (Outbound Port)
- [ ] `app/adapters/persistence/shop_repository.py` 인터페이스 정의:
  ```python
  async def create_shop(session: AsyncSession, tenant_id: UUID, platform: str) -> Shop: ...
  async def get_shop_by_id(session: AsyncSession, shop_id: UUID, tenant_id: UUID) -> Optional[Shop]: ...
  # tenant_id를 조건부로 넘겨 다중 테넌트 열람 금지(Isolation)를 레포지토리 단계부터 강제
  async def update_shop_tokens(session: AsyncSession, shop_id: UUID, token: str, refresh: str, status: str) -> Shop: ...
  ```

### 4. Alembic 연동
- [ ] `alembic revision --autogenerate -m "create_shops_table"`

## 🧪 Acceptance Criteria (BDD/GWT)

**Scenario 1: 토큰 유출 방지 직렬화 검사**
- Given: DB상에 `oauth_token="secret_x"` 값을 보유한 `Shop` 모델 데이터
- When: 내부 API에서 이 인스턴스를 통과시켜 `ShopRead` Pydantic DTO로 변환
- Then: JSON 결과물 안에 `id`, `platform`, `status` 등 메타데이터만 존재하고 `oauth_token` 키 자체가 직렬화 누락된 것을 확인함.

**Scenario 2: 멀티테넌트 Row-Level 획득 방어**
- Given: Tenant B가 엉뚱하게 다른 화주사의 ID(`shop_A_id`) 데이터 질의를 보냄.
- When: 레파지토리의 `get_shop_by_id(session, shop_id=shop_A_id, tenant_id=Tenant_B_id)` 구동
- Then: Where 조건(`shops.id == id AND shops.tenant_id == tenant_id`)의 불일치로 항상 `None`을 반환하여 교차 노출 사고를 물리적으로 차단한다.

## ⚙️ Technical Constraints
- 보안 요건: `oauth_token` 값을 향후 AES-256 저장으로 전환할 여지가 있으므로 컬럼 길이는 Text 등 충분하게 할당한다.

## 🏁 Definition of Done (DoD)
- [ ] `Shop` SQLModel 정의 시 외래키 무결성 적용(`tenants.id`)?
- [ ] 토큰이 생략되는 `ShopRead` DTO 구성 완료?
- [ ] 테넌트 격리를 내재한 `get_shop_by_id(id, tenant_id)` 등의 Repository 인터페이스 파일 작성 완료?
- [ ] Alembic 마이그레이션이 이상없이 Create 쿼리를 던지고 롤백 기능을 지원하는가?
