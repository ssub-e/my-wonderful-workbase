---
name: Feature Task
about: SRS 기반의 구체적인 개발 태스크 명세
title: "[API] ADMIN-004: 팀원 초대 및 Role-Based 권한 부여 API"
labels: 'api, auth, security, phase:4, priority:medium'
assignees: ''
type: task
tags: [task, admin]
created: 2026-04-22
---
## 🎯 Summary
- 기능명: [ADMIN-004] 화주사 사내 새로운 사용자 추가 및 RBAC 롤 맵핑 API
- 목적: 최초로 가입한 회사 대표관리자(ADMIN)가 자사의 실무 센터장이나 물류 담당자(USER) 등 부계정을 할당하고 비밀번호를 부여할 수 있게 한다. `AUTH-001`에서 문서화한 RBAC 체계를 실현하는 관리자 단 API 다.

## 🔗 References (Spec & Context)
- RBAC 명세: `AUTH-001` (권한 체크 가이드라인)
- 보안 규칙: 비밀번호 단방향 해싱(Bcrypt) 적용 의무.

## ✅ Task Breakdown (실행 계획)

### 1. 사용자 신규 생성 DTO
- [ ] `app/schemas/admin_schema.py`:
  ```python
  from pydantic import BaseModel, EmailStr

  class UserInvitationRequest(BaseModel):
      email: EmailStr
      role: str # "SUPERADMIN", "ADMIN", "MD", "CENTER_MANAGER"
      name: str
      temporary_password: str
  ```

### 2. 초대 로직 라우터 구현
- [ ] `app/api/endpoints/admin_users.py`:
  ```python
  from passlib.context import CryptContext
  pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")

  @router.post("/users/invite")
  async def invite_tenant_user(
      payload: UserInvitationRequest,
      tenant_id: str = Depends(get_current_tenant),
      current_user: User = Depends(require_role(["ADMIN"])),
      session: AsyncSession = Depends(get_session)
  ):
      """입력된 임시 비밀번호를 해싱하고 tenant_id를 강제 맵핑하여 신규 유저 생성"""
      
      # 중복 체크
      if await check_email_exists(session, payload.email):
          raise HTTPException(status_code=400, detail="이미 등록된 이메일입니다.")
          
      hashed_pw = pwd_context.hash(payload.temporary_password)
      
      # 부계정 생성 (tenant_id가 현재 접속자와 반드시 같게 고정)
      new_user = await create_user(
          session, 
          tenant_id=tenant_id, 
          email=payload.email, 
          name=payload.name, 
          role=payload.role, 
          hashed_password=hashed_pw
      )
      
      await create_audit_log(session, tenant_id, "CREATE", "USER", new_user.email)
      return {"message": "사용자 생성이 완료되었습니다.", "user_id": new_user.id}
  ```

## 🧪 Acceptance Criteria (BDD/GWT)

**Scenario 1: 권한 위임 체인 탈취 시도 방어**
- Given: `role="MD"` 권한을 가진 직원이 해커 툴을 이용해 `POST /users/invite` 에 관리자 계정을 만들도록 Payload를 던짐.
- When: FastAPI 의존성 `require_role(["ADMIN"])` 이 인터셉트함.
- Then: 403 Forbidden Access 가 뱉어지며 코어 비즈니스 로직에 닿기 전에 원천 차단된다.

**Scenario 2: Tenant 교차 주입 방어 (SSOT)**
- Given: A회사 관리자가 Payload 에 악의적으로 `{"tenant_id": "B"}` 를 끼워넣음.
- When: 신규 유저 생성 모델이 작동함.
- Then: Payload의 `tenant_id`는 철저히 무시되고 오직 현재 발급된 JWT 세션 내부에 하드 코딩된 자기 자신의 `tenant_id`로만 Create가 바인딩 되어 다른 회사에 부계정을 심는 것을 방지한다.

## ⚙️ Technical Constraints
- 추후 `temporary_password` 를 이메일로 자동 송출해주는 메일 서버 연동 방안(SES 등)을 확장할 수 있도록 비즈니스 레이어를 깔끔하게 분리해둔다.

## 🏁 Definition of Done (DoD)
- [ ] Bcrypt 비밀번호 단방향 해싱 구문 탑재 완료?
- [ ] `require_role` 권한 컷오프 의존성 삽입?
- [ ] JWT 추출 ID 기반 맵핑 고정으로 SSRF 급 취약점 방어 완료?
