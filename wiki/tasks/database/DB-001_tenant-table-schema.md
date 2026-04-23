---
name: Feature Task
about: SRS 기반의 구체적인 개발 태스크 명세
title: "[DB] DB-001: TENANT 테이블 스키마 + SQLModel 모델 + Alembic 마이그레이션"
labels: 'database, backend, priority:critical, phase:1'
assignees: ''
type: task
tags: [task, database]
created: 2026-04-22
---
## 🎯 Summary
- 기능명: [DB-001] TENANT 테이블 스키마 + SQLModel 모델 + Alembic 마이그레이션
- 목적: 멀티테넌트 SaaS의 데이터 모델 근간인 TENANT 엔터티를 정의한다. 모든 비즈니스 엔터티(SHOP, FORECAST, REPORT, WORKFORCE_PLAN, AUDIT_LOG)가 tenant_id FK로 이 테이블을 참조하므로, 데이터 모델의 첫 단추이자 SSOT(Single Source of Truth)이다.

## 🔗 References (Spec & Context)
> 💡 AI Agent & Dev Note: 작업 시작 전 아래 문서를 반드시 먼저 Read/Evaluate 할 것.
- SRS ERD: [`SRS-V1.0.md#§6.2`](raw/assets/SRS-V1.0.md) — Entity-Relationship Diagram 전체
- SRS 멀티테넌트: [`SRS-V1.0.md#§4.1.4`](raw/assets/SRS-V1.0.md) — REQ-FUNC-026 tenant_id Row-Level Isolation
- SRS 기술 스택: [`SRS-V1.0.md#§1.5.3`](raw/assets/SRS-V1.0.md) — C-TEC-003 PostgreSQL + SQLModel
- SRS 보안: [`SRS-V1.0.md#§4.2.3`](raw/assets/SRS-V1.0.md) — REQ-NF-018 화주사 데이터 격리
- SRS Stakeholders: [`SRS-V1.0.md#§2`](raw/assets/SRS-V1.0.md) — STK-01~06
- **ERD에서 확인된 TENANT 컬럼:**
  ```
  TENANT {
      uuid id PK
      string name
      string plan_tier
      timestamp created_at
  }
  ```
- **TENANT과의 관계:**
  - TENANT ||--o{ SHOP : "owns"
  - TENANT ||--o{ FORECAST : "generates"
  - TENANT ||--o{ REPORT : "exports"
  - TENANT ||--o{ WORKFORCE_PLAN : "creates"
  - TENANT ||--o{ AUDIT_LOG : "records"

## ✅ Task Breakdown (실행 계획)

### 1. Alembic 초기 설정
- [ ] `alembic init` 실행 및 `alembic.ini` 설정 (sqlalchemy.url → 환경 변수 참조)
- [ ] `env.py`에서 SQLModel 메타데이터 자동 감지 설정
- [ ] `alembic/versions/` 디렉토리 구조 확인

### 2. SQLModel 모델 정의
- [ ] `app/domain/tenant/models.py` — Tenant SQLModel 클래스 정의:
  ```python
  class Tenant(SQLModel, table=True):
      __tablename__ = "tenants"
      
      id: uuid.UUID = Field(default_factory=uuid.uuid4, primary_key=True)
      name: str = Field(max_length=255, nullable=False, description="테넌트(고객사) 이름")
      plan_tier: str = Field(max_length=50, default="free", description="구독 플랜 (free, basic, premium)")
      created_at: datetime = Field(default_factory=datetime.utcnow, description="가입 시각")
      updated_at: Optional[datetime] = Field(default=None, sa_column_kwargs={"onupdate": datetime.utcnow})
      is_active: bool = Field(default=True, description="활성화 상태")
  ```
- [ ] 인덱스 정의: `name` 컬럼에 UNIQUE 제약, `plan_tier`에 일반 인덱스
- [ ] `updated_at` 컬럼 추가 (SRS ERD에 없으나, 운영 추적을 위해 표준 관행으로 추가)
- [ ] `is_active` 컬럼 추가 (소프트 삭제 지원 — SRS ERD에 없으나 멀티테넌트 관리상 필수)

### 3. Pydantic DTO (Request/Response Schema)
- [ ] `app/schemas/tenant.py` — DTO 정의:
  - [ ] `TenantCreate`: 테넌트 생성 요청 (name, plan_tier)
  - [ ] `TenantRead`: 테넌트 조회 응답 (id, name, plan_tier, created_at, is_active)
  - [ ] `TenantUpdate`: 테넌트 수정 요청 (name?, plan_tier?)

### 4. Repository (Persistence Adapter)
- [ ] `app/adapters/persistence/tenant_repository.py` — CRUD 함수:
  - [ ] `create_tenant(session, data: TenantCreate) → TenantRead`
  - [ ] `get_tenant_by_id(session, tenant_id: UUID) → TenantRead | None`
  - [ ] `list_tenants(session, skip, limit) → List[TenantRead]`
  - [ ] `update_tenant(session, tenant_id, data: TenantUpdate) → TenantRead`
  - [ ] `deactivate_tenant(session, tenant_id) → None` (소프트 삭제)

### 5. Alembic 마이그레이션 스크립트
- [ ] `alembic revision --autogenerate -m "create_tenants_table"` 실행
- [ ] 생성된 마이그레이션 스크립트 검증 (upgrade/downgrade 양방향)
- [ ] `alembic upgrade head` 실행 → DB에 테이블 생성 확인

### 6. 시드 데이터
- [ ] 개발용 시드 스크립트 작성 — 테스트 테넌트 2개 생성 (PRD §7-3: 파일럿 고객 2사 가정)
  ```
  Tenant 1: "코스메틱일번지" (plan: "basic") — 김아름(MD) 소속사 시뮬레이션
  Tenant 2: "스피드물류센터" (plan: "basic") — 정동환(센터장) 소속사 시뮬레이션
  ```

## 🧪 Acceptance Criteria (BDD/GWT)

**Scenario 1: 테넌트 테이블 생성**
- Given: PostgreSQL DB가 빈 상태일 때
- When: `alembic upgrade head` 실행
- Then: `tenants` 테이블이 생성되고, `id(PK, UUID)`, `name(VARCHAR, UNIQUE)`, `plan_tier(VARCHAR)`, `created_at(TIMESTAMP)`, `updated_at(TIMESTAMP)`, `is_active(BOOLEAN)` 컬럼이 존재한다.

**Scenario 2: 테넌트 생성 (정상)**
- Given: 유효한 테넌트 정보 `{"name": "코스메틱일번지", "plan_tier": "basic"}`
- When: `create_tenant()` 호출
- Then: DB에 레코드가 저장되고, UUID id가 자동 생성되며, `created_at`이 현재 시간으로 설정된다.

**Scenario 3: 중복 이름 테넌트 생성 시 실패**
- Given: 이미 `name="코스메틱일번지"`인 테넌트가 존재할 때
- When: 동일한 name으로 `create_tenant()` 호출
- Then: `IntegrityError`가 발생하고, 적절한 에러 메시지를 반환한다.

**Scenario 4: 마이그레이션 롤백**
- Given: `alembic upgrade head`로 테이블이 생성된 상태
- When: `alembic downgrade -1` 실행
- Then: `tenants` 테이블이 삭제되고, DB가 이전 상태로 복원된다.

**Scenario 5: 시드 데이터 적재**
- Given: 빈 `tenants` 테이블
- When: 시드 스크립트 실행
- Then: 2개의 테스트 테넌트("코스메틱일번지", "스피드물류센터")가 정상 생성된다.

## ⚙️ Technical & Non-Functional Constraints
- **C-TEC-003 준수**: PostgreSQL + SQLModel 사용, 배포 시 Supabase PostgreSQL
- **REQ-NF-018 준수**: 테넌트 간 데이터 격리의 기반 — 모든 하위 테이블이 `tenant_id FK`를 참조
- **REQ-FUNC-026 준수**: Row-Level Isolation의 핵심 앵커 테이블 — `tenant_id`가 모든 쿼리의 필수 필터 조건
- **UUID v4**: 순차적 ID 대신 UUID 사용 — 멀티테넌트 환경에서 ID 추론 방지 (보안)
- **Alembic**: 마이그레이션은 항상 `autogenerate` + 수동 검토 프로세스를 따름
- **SRS ERD 준수**: ERD에 정의된 4개 컬럼(id, name, plan_tier, created_at) + 운영 확장(updated_at, is_active)

## 🏁 Definition of Done (DoD)
- [ ] 모든 Acceptance Criteria(5개 시나리오)를 충족하는가?
- [ ] SQLModel 모델이 SRS §6.2 ERD의 TENANT 엔터티와 일치하는가?
- [ ] Alembic 마이그레이션 스크립트가 upgrade/downgrade 양방향 동작하는가?
- [ ] Pydantic DTO(Create, Read, Update)가 정의되었는가?
- [ ] Repository CRUD 함수가 구현되었는가?
- [ ] UNIQUE 제약 및 인덱스가 올바르게 설정되었는가?
- [ ] 시드 데이터 스크립트가 작동하는가?
- [ ] pytest 기반 단위 테스트가 작성되고 통과하는가?
- [ ] 코드 리뷰 시 SRS ERD와의 일치성이 확인되었는가?

## 🚧 Dependencies & Blockers
- **Depends on**:
  - ENV-001 (Docker 환경 — PostgreSQL 컨테이너)
  - ENV-006 (FastAPI 기본 구조 — database.py 세션)
- **Blocks**:
  - DB-002 (SHOP 테이블 — `tenant_id FK → tenants.id`)
  - DB-005 (WEATHER_DATA — `tenant_id FK`)
  - DB-006 (TREND_DATA — `tenant_id FK` 유무 확인 필요)
  - DB-007 (FORECAST — `tenant_id FK → tenants.id`)
  - DB-009 (WORKFORCE_PLAN — `tenant_id FK → tenants.id`)
  - DB-010 (AUDIT_LOG — `tenant_id FK → tenants.id`)
  - DB-011 (tenant_id Row-Level Isolation 미들웨어)
  - AUTH-001 (인증 — 테넌트 소속 기반 JWT 발급)
  - AUTH-002 (멀티테넌트 격리 미들웨어)
