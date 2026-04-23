---
name: Feature Task
about: SRS 기반의 구체적인 개발 태스크 명세
title: "[SaaS-Admin] SUPER-002: 최고 관리자 전용 테넌트 통제(가입 승인/정지) API"
labels: 'backend, api, saas-admin, phase:8, priority:high'
assignees: ''
type: task
tags: [task, admin]
created: 2026-04-22
---
## 🎯 Summary
- 기능명: [SUPER-002] SaaS Owner 전용 파트너(Tenant) 제어 라우터 통제 API
- 목적: B2B 시스템 운영사(SaaS 개발사 측)가 영업으로 들어온 파트너의 상태(`ACTIVE`)를 풀어주거나, 결제금 미납 시 사용을 일시 정지(`SUSPENDED`) 시켜버리는 아주 강력한 루트 계정(Root/Super Admin) 전용 기능이다.

## 🔗 References (Spec & Context)
- 변경 대상: `SUPER-001` 의 `TenantStatusEnum`
- 인가(Auth) 정책: `AUTH-001` (Super Admin Role 검증 미들웨어)

## ✅ Task Breakdown (실행 계획)

### 1. 보안 인가 디펜던시 (Super-Admin 전용)
- [ ] `app/api/deps.py`:
  ```python
  from fastapi import Depends, HTTPException
  from app.domain.auth.schema import TokenPayload
  # 이전에 구현된 get_current_user 계통 의존성 재사용

  def get_super_admin(payload: TokenPayload = Depends(verify_jwt)):
      # SaaS 루트 계정인지 확인 (tenant_id가 SYSTEM 이거나 role이 SUPER_ADMIN)
      if payload.role != "SUPER_ADMIN":
          raise HTTPException(status_code=403, detail="SaaS 최고 관리자만 접근 가능합니다.")
      return payload
  ```

### 2. 가입 상태 강제 업데이트 API
- [ ] `app/api/routes/super_admin.py`:
  ```python
  from fastapi import APIRouter, Depends
  from app.api.deps import get_super_admin
  from app.domain.tenant.model import Tenant, TenantStatusEnum, BillingPlanEnum
  
  router = APIRouter(prefix="/saas", tags=["SaaS Owner Control"])

  @router.patch("/tenants/{target_tenant_id}/status", dependencies=[Depends(get_super_admin)])
  async def suspend_or_activate_tenant(target_tenant_id: str, new_status: TenantStatusEnum, db=Depends(get_db)):
      tenant = db.query(Tenant).filter(Tenant.id == target_tenant_id).first()
      if not tenant:
          raise HTTPException(404, "대상을 찾을 수 없습니다.")
          
      tenant.status = new_status
      db.commit()
      return {"message": f"Tenant {target_tenant_id} 의 상태가 {new_status}로 강제 변경되었습니다."}
  ```

## 🧪 Acceptance Criteria (BDD/GWT)

**Scenario 1: 권한 누수(Privilege Escalation) 차단 철벽 검증**
- Given: 평범한 A회사의 관리자(ADMIN 역할)가 로그인 토큰을 손에 튐
- When: Postman 을 켜고 `/api/v1/saas/tenants/TENANT_B/status` 로 `SUSPENDED` 를 날림 
- Then: `get_super_admin` 의존성 검사 통과를 못하고 403 Forbidden 을 맞으며, 어떠한 경우에도 일반 유저가 타사 혹은 시스템 전체의 상태를 조작할 수 없는 견고한 권한 계층이 증명된다.

## ⚙️ Technical Constraints
- 이 API가 타겟팅하는 파라미터는 `SUPER-001` 에서 세팅한 Enum과 정확히 일치하는 Schema 로 바인딩 시켜야 한다. 
- 추후 테넌트가 정지(`SUSPENDED`) 되면 API 전역 미들웨어나 JWT 발급 자체가 멈추어야 SaaS 로서의 요건을 이룬다 (`AUTH-001` 발급체크와 연계).

## 🏁 Definition of Done (DoD)
- [ ] `SUPER_ADMIN` 만 통과시키는 독립적인 의존성(Router Dependency) 코드 작성 완료?
- [ ] 특정 회사의 Status 및 요금제 Plan 을 강제로 바꾸는 PATCH 인터페이스 구현부?
