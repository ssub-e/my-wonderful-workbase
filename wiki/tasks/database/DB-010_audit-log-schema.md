---
name: Feature Task
about: SRS 기반의 구체적인 개발 태스크 명세
title: "[DB] DB-010: AUDIT_LOG 전역 테이블 스키마 + 레파지토리"
labels: 'database, backend, priority:low, phase:1'
assignees: ''
type: task
tags: [task, database]
created: 2026-04-22
---
## 🎯 Summary
- 기능명: [DB-010] 시스템 및 사용자 행위 추적을 위한 AUDIT_LOG 세팅
- 목적: REQ-NF-021(보안/감사 로그)을 준수하여, 다중 테넌트 환경에서 누가 언제 어떤 데이터를 조작했거나 어떤 오류(예측 실패 등)가 났는지를 추적 보관한다. 단순 파일 로깅이 아닌 DB 적재를 통해 관리자 메뉴에서 쉽게 조회할 수 있도록 구성한다.

## 🔗 References (Spec & Context)
- SRS 감사 추적: [`SRS-V1.0.md#§4.2.3`](file:///e:/workspace/SRS-from-PRD/SRS-V1.0.md) — REQ-NF-021 모든 데이터 삭제/수정에 대한 로그 및 6개월 보관 의무.
- SRS ERD: AUDIT_LOG 테이블.
- 관련 제약: GDPR/개인정보 보호를 위해 민감 정보(패스워드 등)는 로깅되지 않아야 함.

## ✅ Task Breakdown (실행 계획)

### 1. SQLModel 스키마 모델링
- [ ] `app/core/logging/models.py` 구성:
  ```python
  from sqlmodel import SQLModel, Field, JSON, Column
  import uuid
  from datetime import datetime
  from typing import Optional, Dict, Any

  class AuditLog(SQLModel, table=True):
      __tablename__ = "audit_logs"
      
      id: uuid.UUID = Field(default_factory=uuid.uuid4, primary_key=True)
      tenant_id: Optional[uuid.UUID] = Field(foreign_key="tenants.id", index=True, nullable=True) # 시스템 백그라운드 에러의 경우 Null 허용
      user_id: Optional[uuid.UUID] = Field(index=True, nullable=True) # 조작 주체
      action: str = Field(max_length=50, index=True, description="CREATE, UPDATE, DELETE, PREDICT_FAIL 등")
      entity_type: str = Field(max_length=50, description="ex: ORDER, FORECAST, SHOP_TOKEN")
      entity_id: str = Field(max_length=255, description="조작 대상의 ID 문자열")
      changes: Optional[Dict[str, Any]] = Field(default=None, sa_column=Column(JSON)) # 이전값/이후값 JSON 보관
      ip_address: Optional[str] = Field(max_length=45, default=None)
      created_at: datetime = Field(default_factory=datetime.utcnow, index=True)
  ```

### 2. 읽기 전용 DTO
- [ ] `app/schemas/log_schema.py`
  - `AuditLogRead`: 관리자 뷰 응답용. `changes` 딕셔너리의 재귀적 직렬화 검토.

### 3. Repository 오퍼레이션
- [ ] `app/adapters/persistence/audit_repository.py`:
  ```python
  # HTTP 미들웨어나 서비스 레이어에서 비동기로 훅킹 될 로깅 함수 (넌블로킹 강제)
  async def create_audit_log(session: AsyncSession, log_data: AuditLog) -> None: ...
  
  # 조회 (어드민 / 혹은 화주사 최고관리자용)
  async def get_audit_logs(session: AsyncSession, tenant_id: UUID, skip: int=0, limit: int=50) -> List[AuditLog]: ...
  ```

## 🧪 Acceptance Criteria (BDD/GWT)

**Scenario 1: JSON 형태의 조작 내역(changes) 저장**
- Given: 비즈니스 레이어에서 특정 상품의 수량이 10 -> 5 로 변하는 UPDATE 트랜잭션 발생
- When: 로깅 미들웨어가 캡쳐하여 `changes = {"before": {"qty": 10}, "after": {"qty": 5}}` 형태를 묶어 `create_audit_log` 호출.
- Then: PostgreSQL `JSONB` 형식으로 온전히 저장되며 차후 조회 시 Pydantic `Dict` 타입으로 자동 파싱되어야 한다.

**Scenario 2: 대량 조회 속도 보장 (인덱스 검증)**
- Given: 수 백만건의 누적 로그 (REQ-NF-021 6개월 보관 규정)
- When: 특정 테넌트가 특정 `action='DELETE'` 내역만을 `created_at` 최신순으로 조회
- Then: `tenant_id`, `action`, `created_at` 에 걸린 색인(Index) 덕분에 응답이 100ms 이내로 도출된다.

## ⚙️ Technical Constraints
- SQLAlchemy `JSON` 속성을 이용하여 Postgres `JSONB` 타입의 이점을 극대화 시킬 것 (내부 스키마 검색 속도 최적화 가능성).
- 모든 HTTP 변경 요청에 대해 이 함수를 호출할 경우 API Latency가 증가할 수 있으므로, FastAPI BackgroundTasks 를 이용하여 비동기 위임 처리하는 방식을 권장.

## 🏁 Definition of Done (DoD)
- [ ] Log 스키마 내에 Tenant 추적 외래키 및 JSON 컬럼 생성 완료?
- [ ] BackgroundTasks 연동을 고려한 Repo 구축?
- [ ] Alembic Revision 생성 성공?
