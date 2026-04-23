---
name: Feature Task
about: SRS 기반의 구체적인 개발 태스크 명세
title: "[DB] DB-009: 인원 역산 (WORKFORCE_PLAN) 테이블 + SQLModel 모델"
labels: 'database, backend, priority:high, phase:1'
assignees: ''
type: task
tags: [task, database]
created: 2026-04-22
---
## 🎯 Summary
- 기능명: [DB-009] WORKFORCE_PLAN 인력 산출 및 알림 이력 테이블 생성
- 목적: 수요 예측(FORECAST)을 가공하여 3PL 센터에서 필요한 익일 필요 적정 아르바이트 인원 수를 역산해 저장하고, 알림톡 통보 이력(시간, 매체)을 추적 및 보존한다 (REQ-FUNC-019, REQ-FUNC-024 대응).

## 🔗 References (Spec & Context)
- SRS 다이어그램: [`SRS-V1.0.md#§6.2`](raw/assets/SRS-V1.0.md) — WORKFORCE_PLAN
- 인원 역산 기능: [`SRS-V1.0.md#§4.1.3`](raw/assets/SRS-V1.0.md) — REQ-FUNC-019 (적정 인원 자동 역산), REQ-FUNC-024 (알림 발송 감사 로그 및 통보 기록)

## ✅ Task Breakdown (실행 계획)

### 1. 역산 엔진 스키마 포트
- [ ] `app/domain/workforce/models.py` 구성:
  ```python
  class WorkforcePlan(SQLModel, table=True):
      __tablename__ = "workforce_plans"
      
      id: uuid.UUID = Field(default_factory=uuid.uuid4, primary_key=True)
      tenant_id: uuid.UUID = Field(foreign_key="tenants.id", index=True, nullable=False)
      center_id: str = Field(max_length=50, index=True, description="창고/물류센터 식별자")
      target_date: date = Field(nullable=False, description="알바생이 필요한 익일 등의 미래 일자")
      
      # 산출 근거 필드
      predicted_shipments: int = Field(default=0, description="총 예상 출고 건 (합산 조인 쿼리 결과치)")
      capacity_per_worker: float = Field(default=50.0, description="1인당 CAPA 처리량")
      recommended_workers: int = Field(default=0, description="최종 산출된 역산 권고 인력 (수동조정 포함)")
      
      # 발송 관리
      notification_channel: Optional[str] = Field(default=None, description="kakaotalk, sms_fallback 등")
      notified_at: Optional[datetime] = Field(default=None, description="알림톡 발송 성공 시각 저장")
  ```

### 2. 읽기 및 조작 DTO 객체
- [ ] `app/schemas/workforce_schema.py`:
  - `PlanGenerateRequest`: 역산 로직 트리거를 외부에서 수동 처리시킬 경우의 인풋 스키마
  - `PlanRead`: 프론트엔드 대시보드의 표식 렌더링에 적합한 데이터 필터 체계

### 3. Repository 인터페이스
- [ ] `app/adapters/persistence/workforce_repository.py`:
  ```python
  # 역산치 커밋 (인원 저장)
  async def save_workforce_plan(session: AsyncSession, plan: WorkforcePlan) -> WorkforcePlan: ...
  
  # 오후 16시 알림톡 스케줄러가 인출할 당일 분량 플랜 (배치용)
  async def get_plans_pending_notification(session: AsyncSession, target_date: date) -> List[WorkforcePlan]: ...
  
  # 발송 성공 후 매체 시간 타임스탬프 스탬핑 (REQ-FUNC-024)
  async def update_notified_status(session: AsyncSession, plan_id: UUID, channel: str) -> None: ...
  ```

### 4. 마이그레이션 적용 (Alembic)
- [ ] `alembic revision --autogenerate -m "create_workforce_plan_table"`

## 🧪 Acceptance Criteria (BDD/GWT)

**Scenario 1: 적정 인원 자동 스케줄 산출 저장**
- Given: 익일 기준의 예측 출고량(predicted_shipments) 1,500건으로 요약 데이터가 주어짐
- When: 역산 비즈니스 계층이 `save_workforce_plan` 함수로 데이터를 보냄. (인당 50 가정하여 권장 인원=30 도출됨)
- Then: 모델 유효성에 맞아 `recommended_workers=30`과 원본 출고량 예측치가 통계 손실 없이 DB에 저장된다.

**Scenario 2: 통보 이력 및 타임스태핑 (REQ-FUNC-024)**
- Given: 위 생성된 데이터 중 빈 `notified_at` 컬럼 상태.
- When: 16시 스케줄러가 Kakao 알림톡을 성공시키고 `update_notified_status(..., "kakaotalk")`를 호출함.
- Then: DB 내 `notification_channel` 컬럼과 `notified_at` 값이 현재 시간부로 채워지고, 이후 재발송시 중복 필터링 용도로 사용할 수 있게 된다.

## ⚙️ Technical Constraints
- 필드 타입 제약: `recommended_workers` (권장 인원)이나 `predicted_shipments` (출고량) 등은 이산적 개체이므로 파이썬의 `float` 타입이 들어오더라도 `math.ceil` 연산을 통한 `integer` 타입 유지가 필요하다.

## 🏁 Definition of Done (DoD)
- [ ] 모델 내에 산출 근거 3종과 발송 관리 필드 2종이 모두 포함되었나?
- [ ] 인원에 대한 `INT` 자료형과 외래키(`tenant_id`) 제약 지정 완료?
- [ ] 오후 16시 배치 봇이 읽어들일 용도의 Repository Fetch 인터페이스 구현됨?
