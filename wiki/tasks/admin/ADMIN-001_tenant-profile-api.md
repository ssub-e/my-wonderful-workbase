---
name: Feature Task
about: SRS 기반의 구체적인 개발 태스크 명세
title: "[API] ADMIN-001: 화주사(Tenant) 기본 프로필 관리 라우터"
labels: 'api, admin, phase:4, priority:medium'
assignees: ''
type: task
tags: [task, admin]
created: 2026-04-22
---
## 🎯 Summary
- 기능명: [ADMIN-001] 화주사 정보 조회 및 수정 API
- 목적: SaaS를 구독한 고객(Tenant)이 자신의 백오피스 환경설정 탭에서 회사명, 사업자번호, 대표 알림톡 수신 번호(`admin_phone`) 등을 확인하고 직접 수정할 수 있는 권한을 제공한다.

## 🔗 References (Spec & Context)
- DB 모델 연계: `DB-001` (TENANT 스키마)
- 알림톡 기능 연계: `ADAPT-004` (여기에 저장된 전화번호로 알림이 감)

## ✅ Task Breakdown (실행 계획)

### 1. 프로필 DTO 스펙
- [ ] `app/schemas/admin_schema.py`:
  ```python
  from pydantic import BaseModel, constr

  class TenantProfileUpdate(BaseModel):
      company_name: str
      admin_phone: constr(pattern=r"^010\d{8}$") # 정규식 기반 휴대폰 번호 검증
      contact_email: str
  
  class TenantProfileResponse(TenantProfileUpdate):
      tenant_id: str
      created_at: str
      subscription_status: str
  ```

### 2. FastAPI 라우터 및 레포지토리 연결
- [ ] `app/api/endpoints/admin.py`:
  ```python
  @router.get("/profile", response_model=TenantProfileResponse)
  async def get_tenant_profile(
      tenant_id: str = Depends(get_current_tenant),
      session: AsyncSession = Depends(get_session)
  ):
      # 단순 Tenant 테이블 조회 리턴
      tenant = await get_tenant(session, tenant_id)
      return tenant

  @router.put("/profile", response_model=TenantProfileResponse)
  async def update_tenant_profile(
      payload: TenantProfileUpdate,
      tenant_id: str = Depends(get_current_tenant),
      session: AsyncSession = Depends(get_session)
  ):
      tenant = await update_tenant_info(session, tenant_id, payload.model_dump())
      
      # [AUDIT] 사용자 액션 로그 기록 (미들웨어 또는 명시적 호출)
      await create_audit_log(session, tenant_id, "UPDATE", "TENANT_PROFILE", str(tenant_id))
      return tenant
  ```

## 🧪 Acceptance Criteria (BDD/GWT)

**Scenario 1: 알림톡 번호 포맷 밸리데이션(Pydantic)**
- Given: 고객이 알림톡을 받을 번호로 `02-1234-5678` 이나 `010-1234-5678`(하이픈 포함)을 입력함
- When: `PUT /admin/profile` 호출
- Then: Pydantic의 `constr(pattern)` 규칙 위반으로 인해 422 Unprocessable Entity 에러가 반환되어, 하이픈 없는 순수 010 번호만이 DB에 저장되도록 강제함으로써 카카오톡 발송기(ADAPT-004)의 런타임 에러를 1차 예방한다.

## ⚙️ Technical Constraints
- 오직 권한 롤이 `ADMIN` 인 사용자만이 호출할 수 있도록, 의존성 `get_current_tenant` 내부에 Role-check 로직이 포함되거나 `@require_role('admin')` 등의 커스텀 데코레이터를 부착해야 한다.

## 🏁 Definition of Done (DoD)
- [ ] 프로필 조회 및 업데이트 PUT/GET 라우터 작성 완료?
- [ ] 연락처 정규식 검증 규칙 DTO 적용 확인?
- [ ] 프로필 변경 시 Audit Log 연동 기록 확인?
