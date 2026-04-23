---
name: Feature Task
about: SRS 기반의 구체적인 개발 태스크 명세
title: "[Feature/Command] AUTH-001: OAuth 2.0 + JWT 사용자 인증 + RBAC 접근 제어 구현"
labels: 'feature, backend, security, priority:critical, phase:1'
assignees: ''
type: task
tags: [task, security]
created: 2026-04-22
---
## 🎯 Summary
- 기능명: [AUTH-001] OAuth 2.0 + JWT 사용자 인증 + RBAC 접근 제어 구현
- 목적: SaaS 사용자(이커머스 MD, 3PL 센터장, 시스템 관리자)가 안전하게 인증하고, 역할별 접근 권한에 따라 시스템 기능을 이용할 수 있도록 한다. 멀티테넌트 환경에서 모든 API 요청에 JWT 토큰을 통한 인증 + 테넌트 식별 + 역할 기반 접근 제어를 강제한다.

## 🔗 References (Spec & Context)
> 💡 AI Agent & Dev Note: 작업 시작 전 아래 문서를 반드시 먼저 Read/Evaluate 할 것.
- SRS 인증: [`SRS-V1.0.md#§4.1.4`](raw/assets/SRS-V1.0.md) — REQ-FUNC-025 (OAuth 2.0 + JWT, RBAC)
- SRS 멀티테넌트: [`SRS-V1.0.md#§4.1.4`](raw/assets/SRS-V1.0.md) — REQ-FUNC-026 (tenant_id Row-Level Isolation)
- SRS 보안 NFR: [`SRS-V1.0.md#§4.2.3`](raw/assets/SRS-V1.0.md) — REQ-NF-017 (인증 및 접근 제어)
- SRS 아키텍처: [`SRS-V1.0.md#§3.6`](raw/assets/SRS-V1.0.md) — Component Architecture "인증/인가 OAuth 2.0 + JWT + RBAC"
- SRS 감사 로그: [`SRS-V1.0.md#§4.1.4`](raw/assets/SRS-V1.0.md) — REQ-FUNC-028 (감사 로그 기록)
- SRS Stakeholders: [`SRS-V1.0.md#§2`](raw/assets/SRS-V1.0.md) — 역할 정의 (MD, 센터장, 경영진, 화주사, 시스템 관리자)
- SRS Traceability: [`SRS-V1.0.md#§5.1`](raw/assets/SRS-V1.0.md) — TC-025: RBAC 접근 제어 검증

## ✅ Task Breakdown (실행 계획)

### 1. 사용자 모델 (User Entity)
- [ ] `app/domain/auth/models.py` — User SQLModel 정의:
  ```python
  class User(SQLModel, table=True):
      __tablename__ = "users"
      
      id: uuid.UUID = Field(default_factory=uuid.uuid4, primary_key=True)
      tenant_id: uuid.UUID = Field(foreign_key="tenants.id", nullable=False, index=True)
      email: str = Field(max_length=255, unique=True, nullable=False)
      hashed_password: str = Field(nullable=False)
      name: str = Field(max_length=100, nullable=False)
      role: str = Field(max_length=50, nullable=False)  # md, center_manager, admin, shipper
      is_active: bool = Field(default=True)
      created_at: datetime = Field(default_factory=datetime.utcnow)
      last_login_at: Optional[datetime] = Field(default=None)
  ```
- [ ] Alembic 마이그레이션: `create_users_table`
- [ ] 역할 Enum 정의:
  ```python
  class UserRole(str, Enum):
      MD = "md"                      # 이커머스 MD (STK-01)
      CENTER_MANAGER = "center_manager"  # 3PL 센터장 (STK-02)
      ADMIN = "admin"                # 시스템 관리자 (STK-06)
      SHIPPER = "shipper"            # 화주사 담당자 (STK-04)
  ```

### 2. JWT 토큰 생성/검증
- [ ] `app/core/security.py` — JWT 유틸리티:
  - [ ] `create_access_token(user_id, tenant_id, role, expires_delta)` → JWT 토큰 생성
  - [ ] `create_refresh_token(user_id, tenant_id)` → 리프레시 토큰 생성
  - [ ] `verify_token(token)` → 디코딩 + 유효성 검증 → 페이로드 반환
  - [ ] JWT 페이로드 구조:
    ```json
    {
      "sub": "user_uuid",
      "tenant_id": "tenant_uuid",
      "role": "md",
      "exp": 1234567890,
      "iat": 1234567800
    }
    ```
  - [ ] Access Token 유효기간: 30분
  - [ ] Refresh Token 유효기간: 7일

### 3. 비밀번호 해싱
- [ ] `passlib[bcrypt]` 기반 단방향 비밀번호 해싱
- [ ] `hash_password(plain) → hashed`
- [ ] `verify_password(plain, hashed) → bool`
- [ ] ⚠️ **비밀번호 평문 저장 절대 금지** (REQ-NF-016)

### 4. 인증 API 엔드포인트
- [ ] `app/api/v1/auth.py` — 인증 라우터:
  - [ ] `POST /api/v1/auth/login` — 이메일 + 비밀번호 → JWT 토큰 발급
  - [ ] `POST /api/v1/auth/refresh` — Refresh Token → 새 Access Token 발급
  - [ ] `GET /api/v1/auth/me` — 현재 인증된 사용자 정보 조회
  - [ ] `POST /api/v1/auth/logout` — 토큰 블랙리스트 등록 (Redis)

### 5. RBAC 미들웨어 및 의존성 주입
- [ ] `app/middleware/auth.py` — JWT 인증 미들웨어:
  - [ ] `get_current_user(token: str = Depends(oauth2_scheme))` → User 객체 반환
  - [ ] `get_current_active_user(user = Depends(get_current_user))` → 활성 사용자 확인
- [ ] `app/core/rbac.py` — 역할 기반 접근 제어:
  - [ ] `require_role(*allowed_roles)` — 데코레이터 팩토리
  ```python
  # 사용 예시
  @router.get("/admin/logs")
  async def get_audit_logs(user = Depends(require_role(UserRole.ADMIN))):
      ...  # ADMIN 역할만 접근 가능
  ```
  - [ ] 역할별 접근 매트릭스 정의:
    | 기능 | MD | CENTER_MANAGER | ADMIN | SHIPPER |
    |---|---|---|---|---|
    | 리포트 생성/조회 | ✅ | ❌ | ✅ | ❌ |
    | 대시보드 접속 | ✅ | ✅ | ✅ | ❌ |
    | 쇼핑몰 연동 | ❌ | ❌ | ✅ | ✅ |
    | 인원 대시보드 | ❌ | ✅ | ✅ | ❌ |
    | 감사 로그 조회 | ❌ | ❌ | ✅ | ❌ |

### 6. 인증 이벤트 감사 로그
- [ ] 로그인 성공/실패, 토큰 갱신, 로그아웃 이벤트를 AUDIT_LOG에 기록 (REQ-FUNC-028 연계)

## 🧪 Acceptance Criteria (BDD/GWT)

**Scenario 1: 정상 로그인 → JWT 발급**
- Given: 유효한 이메일(`md@cosmeticfirst.com`)과 올바른 비밀번호가 주어짐
- When: `POST /api/v1/auth/login` 요청
- Then: `200 OK` + `{"access_token": "...", "refresh_token": "...", "token_type": "bearer"}` 반환. JWT 페이로드에 `sub`, `tenant_id`, `role`이 포함됨.

**Scenario 2: 잘못된 비밀번호 로그인 시도**
- Given: 등록된 이메일이지만 잘못된 비밀번호가 주어짐
- When: `POST /api/v1/auth/login` 요청
- Then: `401 Unauthorized` + `{"error": {"code": "INVALID_CREDENTIALS", "message": "이메일 또는 비밀번호가 잘못되었습니다"}}` 반환. 감사 로그에 실패 기록.

**Scenario 3: 만료된 토큰으로 API 접근**
- Given: 유효기간이 만료된 Access Token
- When: 해당 토큰으로 인증이 필요한 API 요청
- Then: `401 Unauthorized` + `{"error": {"code": "TOKEN_EXPIRED", "message": "..."}}` 반환.

**Scenario 4: Refresh Token으로 Access Token 갱신**
- Given: 유효한 Refresh Token
- When: `POST /api/v1/auth/refresh` 요청
- Then: 새로운 Access Token이 발급되고, 기존 Refresh Token은 교체(Rotate)된다.

**Scenario 5: RBAC — 권한 없는 역할의 접근 차단**
- Given: 역할이 `shipper`인 사용자의 유효한 JWT
- When: `GET /api/v1/dashboard/forecast` (MD/센터장 전용) 요청
- Then: `403 Forbidden` + `{"error": {"code": "INSUFFICIENT_PERMISSIONS", "message": "해당 리소스에 접근 권한이 없습니다"}}` 반환.

**Scenario 6: RBAC — 감사 로그 관리자 전용 접근**
- Given: 역할이 `admin`인 사용자의 유효한 JWT
- When: 감사 로그 조회 API 요청
- Then: `200 OK` + 감사 로그 목록 반환.

**Scenario 7: 비활성 사용자 접근 차단**
- Given: `is_active=false`인 사용자의 유효한 JWT
- When: 인증이 필요한 API 요청
- Then: `403 Forbidden` + `{"error": {"code": "ACCOUNT_DISABLED", "message": "..."}}` 반환.

## ⚙️ Technical & Non-Functional Constraints
- **REQ-FUNC-025 준수**: OAuth 2.0 + JWT 인증, RBAC 적용
- **REQ-NF-017 준수**: OAuth 2.0 + JWT, RBAC 보안 기준
- **REQ-NF-015 준수**: TLS 1.3 위에서 토큰 전송 (SEC-001 선행 필요)
- **REQ-NF-016 준수**: 비밀번호 Bcrypt 해싱 (평문 저장 절대 금지)
- **REQ-NF-019 연계**: 인증 이벤트 감사 로그 기록
- **성능**: 로그인 API 응답 시간 p95 ≤ 500ms
- **보안**:
  - JWT Secret Key는 환경 변수로 관리 (하드코딩 금지)
  - 로그에 토큰, 비밀번호 평문 노출 절대 금지
  - Refresh Token Rotation 적용 (재사용 감지 시 전체 세션 무효화)
- **라이브러리**: `python-jose[cryptography]`, `passlib[bcrypt]`, `python-multipart`

## 🏁 Definition of Done (DoD)
- [ ] 모든 Acceptance Criteria(7개 시나리오)를 충족하는가?
- [ ] JWT 토큰 생성/검증이 정상 동작하는가?
- [ ] Bcrypt 비밀번호 해싱/검증이 정상 동작하는가?
- [ ] RBAC 역할별 접근 제어 매트릭스가 구현/검증되었는가?
- [ ] Refresh Token Rotation이 구현되었는가?
- [ ] 인증 이벤트가 AUDIT_LOG에 기록되는가?
- [ ] 단위 테스트 + 통합 테스트가 작성되고 통과하는가? (커버리지 ≥ 80%)
- [ ] Swagger/OpenAPI 문서에 인증 스키마가 반영되었는가?
- [ ] 비밀번호/토큰이 로그에 노출되지 않는가?

## 🚧 Dependencies & Blockers
- **Depends on**:
  - DB-001 (TENANT 테이블 — `users.tenant_id FK → tenants.id`)
  - ENV-005 (환경 변수 관리 — SECRET_KEY, JWT 설정)
  - ENV-006 (FastAPI 기본 구조 — security.py, 미들웨어 프레임)
- **Blocks**:
  - AUTH-002 (멀티테넌트 tenant_id Row-Level Isolation 미들웨어 — JWT에서 tenant_id 추출)
  - F2-W01 (카페24 OAuth 연동 — 인증 기반 필요)
  - SEC-003 (OAuth 2.0 + JWT + RBAC 보안 검증 — 검증 대상)
  - TEST-025 (RBAC 접근 제어 GWT 검증 테스트)
  - 모든 인증이 필요한 API 엔드포인트
