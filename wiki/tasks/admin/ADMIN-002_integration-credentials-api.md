---
name: Feature Task
about: SRS 기반의 구체적인 개발 태스크 명세
title: "[API] ADMIN-002: 쇼핑몰 연동 식별키(Credentials) 등록 API"
labels: 'api, admin, security, phase:4, priority:critical'
assignees: ''
type: task
tags: [task, admin]
created: 2026-04-22
---
## 🎯 Summary
- 기능명: [ADMIN-002] 타겟 쇼핑몰 자격증명 보안 등록 라우터
- 목적: 카페24 등 화주사가 사용하는 쇼핑몰의 `mall_id` 와 임시 `app_key` 등을 SaaS 시스템에 건네주어 OAuth 인증을 인가받게 하는 진입점 API. 암호화 적용이 필수적이다.

## 🔗 References (Spec & Context)
- SRS 외부망 요구사항: [`SRS-V1.0.md#§3.1`](raw/assets/SRS-V1.0.md) — EXT-01 (쇼핑몰 호스팅 연동)
- 관련 스키마: `DB-002` (SHOP 테이블)

## ✅ Task Breakdown (실행 계획)

### 1. 양방향 암호화 유틸리티 작성
- [ ] `app/core/security/crypto.py`:
  - `ENV-005` 의 `SECRET_KEY` 를 대칭키로 이용하는 `cryptography.fernet` 등 단순 양방향 암호화 모듈 작성. (Refresh 토큰은 나중에 원본 텍스트로 꺼내어 써야 하므로 해싱(Bcrypt)이 아닌 양방향 암호화를 써야함).

### 2. 쇼핑몰 인증 등록 API (OAuth 초기 연동)
- [ ] `app/api/endpoints/admin.py`:
  ```python
  from app.core.security.crypto import encrypt_token

  class ShopCredentialPayload(BaseModel):
      platform_type: str = "cafe24"
      platform_mall_id: str
      auth_code: str # 사용자가 팝업 등에서 받아온 최초 인증 코드
      
  @router.post("/integrations/shop")
  async def connect_shop_integration(
      payload: ShopCredentialPayload,
      tenant_id: str = Depends(get_current_tenant),
      session: AsyncSession = Depends(get_session)
  ):
      # 1. 전달받은 auth_code 로 카페24 본 서버와 통신하여 영구 Refresh / Access 토큰을 발급 (ADAPT-003 기능 활용)
      tokens = await Cafe24Adapter.exchange_auth_code(payload.platform_mall_id, payload.auth_code)
      
      # 2. 토큰 양방향 암호화 방어 (평문 DB 적재 금지 룰 충족)
      safe_refresh = encrypt_token(tokens['refresh_token'])
      safe_access = encrypt_token(tokens['access_token'])
      
      # 3. DB-002의 SHOP 테이블에 레코드 INSERT
      new_shop = await create_shop(session, tenant_id, payload.platform_mall_id, safe_refresh, safe_access)
      
      return {"message": "쇼핑몰 연동이 완료되었습니다.", "shop_id": new_shop.id}
  ```

## 🧪 Acceptance Criteria (BDD/GWT)

**Scenario 1: 크리덴셜(토큰) 암호화 저장 확인**
- Given: 사용자가 유효한 `auth_code`를 서버로 넘겨 연동을 승인함
- When: 백엔드가 외부 서버로부터 `refreshToken_abc123` 을 발급받고 DB에 적재함
- Then: DB 관리자가 `SELECT * FROM shops` 를 수행하더라도, `oauth_token` 컬럼에는 원본 텍스트 대신 `gAAAAABkXYZ...` 형태의 알아볼 수 없는 Fernet 암호화 값만이 저장되어 있어, DB 탈취 사고 시 화주사의 쇼핑몰 지배권이 뺏기는 것을 방지한다.

## ⚙️ Technical Constraints
- OAuth 2.0 흐름상 프론트엔드가 Redirect URI로 `auth_code`를 받는 플로우를 전제로 기획되었다. 이 액션이 끝나면 즉시 수집 Worker(`F2-W03`)가 반응해야 하므로, 등록 완료 후 트리거를 주거나 다음날 아침부터 Batch가 수집하게끔 유도해야 한다.

## 🏁 Definition of Done (DoD)
- [ ] Fernet 등 양방향 대칭키 암호화 함수 작성 완료?
- [ ] `auth_code` 기반 토큰 교환 및 암호화/저장 DB 로직 구성 완료?
