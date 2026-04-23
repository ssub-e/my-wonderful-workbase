---
name: Feature Task
about: SRS 기반의 구체적인 개발 태스크 명세
title: "[SaaS-Admin] SUPER-001: SaaS 과금(Billing) 및 테넌트(화주사) 상태 테이블 확장"
labels: 'backend, db, saas-admin, phase:8, priority:high'
assignees: ''
type: task
tags: [task, admin]
created: 2026-04-22
---
## 🎯 Summary
- 기능명: [SUPER-001] Tenant 메타데이터를 이용 정지 및 요금제 기반으로 확장
- 목적: B2B SaaS의 필수 요건인 "돈을 안내면 기능을 차단하거나, 프리미엄(Pro) 요금제에만 고급 AI를 태워주기" 기능을 위해 `DB-001` 의 TENANT 객체를 고도화 시킨다.

## 🔗 References (Spec & Context)
- 대상 스키마: `DB-001` (`Tenant` 모델)

## ✅ Task Breakdown (실행 계획)

### 1. 필드 추가 (SQLModel)
- [ ] `app/domain/tenant/model.py`:
  ```python
  from sqlmodel import SQLModel, Field
  from datetime import datetime
  from enum import Enum
  
  class TenantStatusEnum(str, Enum):
      ACTIVE = "ACTIVE"
      SUSPENDED = "SUSPENDED"  # 미납/계약종료 등으로 이용 정지
      TRIAL = "TRIAL"          # 한달 무료 체험
      
  class BillingPlanEnum(str, Enum):
      FREE = "FREE"            # 기본 API만 
      PRO = "PRO"              # AI 예측 포함
      ENTERPRISE = "ENTERPRISE" # XAI/커스텀 도메인 포함

  class Tenant(SQLModel, table=True):
      id: str = Field(primary_key=True)
      company_name: str
      ... # 기존 컬럼들
      
      # 신규 필드
      status: TenantStatusEnum = Field(default=TenantStatusEnum.TRIAL)
      plan: BillingPlanEnum = Field(default=BillingPlanEnum.FREE)
      trial_ends_at: datetime | None = Field(default=None)
  ```

### 2. Alembic 마이그레이션 스크립트 작성
- [ ] `alembic revision --autogenerate -m "add_billing_and_status_to_tenant"` 파생 및 적용. (기존 테넌트들은 `ACTIVE` 에 `PRO` 로 Migration Fallback).

## 🧪 Acceptance 초도 기준 (BDD/GWT)

**Scenario 1: Enum 파싱 무결성**
- Given: SQLModel 에서 `Enum` 으로 타겟팅된 `status` 필드
- When: 누군가 관리자 API를 통해 `status="BLOCKED"` 라는 미승인 텍스트를 PATCH 로 밀어넣으려 함.
- Then: Pydantic 의 백그라운드 제어망에 의해 422 Unprocessable Entity 에러가 나야 하며, 데이터베이스 `Varchar` 컬럼에 Enum 범위를 이탈한 쓰레기값이 들어가 애플리케이션 수명주기를 파괴하는 사고를 방어한다.

## ⚙️ Technical Constraints
- SaaS 상태 변경 컬럼(`status`, `plan`) 은 절대 일반 어드민(`ADMIN-xxx`)이나 유저가 자기 자신의 데이터를 수정(`PATCH`) 하도록 열어주면 안된다. 오직 시스템(System) 백그라운드나 최고운영자 권한 라우터에서만 건드려야 함(보안 격리).

## 🏁 Definition of Done (DoD)
- [ ] Enum(상태, 요금제) 기반 데이터 모델 Field 추가 확정?
- [ ] DB 자동 생성(Alembic) 명세 반영?
