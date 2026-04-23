---
name: Feature Task
about: SRS 기반의 구체적인 개발 태스크 명세
title: "[Feature] SEC-008: 동시 접속 제어 및 활성 디바이스 강제 로그아웃 관리 UI"
labels: 'feature, backend, security, priority:low'
assignees: ''
type: task
tags: [task, security]
created: 2026-04-22
---
## 🎯 Summary
- 기능명: [SEC-008] 사용자 세션 통합 관리 및 보안 제어
- 목적: 계정 공유 및 무단 접근 방지를 강화한다. 유저가 현재 로그인되어 있는 모든 기기 목록(브라우저 종류, 위치, 접속 시간)을 보여주고, 분실된 기기나 의심스러운 환경에서 접속된 세션을 원격에서 즉시 '강제 로그아웃' 시킬 수 있는 보안 기능을 제공한다.

## 🔗 References (Spec & Context)
> 💡 AI Agent & Dev Note: 작업 시작 전 아래 문서를 반드시 먼저 Read/Evaluate 할 것.
- 세션 저장소: `INFRA-002` (Redis)
- 인증 토큰: `AUTH-001` (JWT Refresh Token)
- 감사 로그: `DB-010` (Audit Log)

## ✅ Task Breakdown (실행 계획)
- [ ] **Session Tracker** 구현:
  - 로그인 시 `User-Agent` 와 `IP` 기반 지리 정보(GeoIP)를 추출하여 Redis 에 `session:{user_id}:{token_jti}` 키로 저장
- [ ] **Active Sessions API**: 현재 로그인 중인 모든 활성 세션 목록 조회 엔드포인트 구축
- [ ] **Remote Logout Action**: 특정 세션 식별자(`jti`)를 전달받아 Redis 에서 즉시 삭제 및 Blacklist 등록 로직
- [ ] **Login Policy**: 테넌트별 '동시 접속 가능 기기 수' 제한 정책 옵션 적용
- [ ] 프론트엔드: '마이 프로필' 내 '현재 로그인 정보' 탭 구축 및 기기별 '연결 끊기' 버튼 컴포넌트

## 🧪 Acceptance Criteria (BDD/GWT)
Scenario 1: 실시간 활성 기기 목록 조회
- Given: 크롬 브라우저와 모바일 앱으로 동시에 로그인한 유저
- When: 보안 설정 탭 진입
- Then: "Windows - Chrome (서울, 현재 접속 중)", "iPhone - App (공항동, 2시간 전)" 2건의 리스트가 출력된다.

Scenario 2: 타 기기 원격 로그아웃
- Given: PC에서 로그인된 상태로 모바일을 분실한 유저
- When: PC 화면에서 모바일 세션 옆 '연결 끊기' 클릭
- Then: 즉시 해당 모바일 앱에서는 API 호출 시 `401 Unauthorized` 가 발생하며 토큰이 만료되어 로그아웃 처리된다.

## ⚙️ Technical & Non-Functional Constraints
- 성능: 로그인 시마다 Redis 에 정보를 쓰므로 Latency 에 유의 (Set-and-Forget 처리)
- 보안: `jti` (JWT Unique ID)를 활용하여 특정 토큰만 정밀하게 타격하여 만료 처리
- UX: '현재 내가 접속 중인 이 기기' 는 별도 강조 표시 및 "모든 다른 기기에서 로그아웃" 기능 필수 제공

## 💻 Implementation Snippet (Redis Session Management)
```python
async def register_session(user_id: str, jti: str, user_agent: str, ip: str):
    session_info = {
        "device": parse_ua(user_agent),
        "ip": ip,
        "last_active": datetime.now().isoformat()
    }
    # User-Session Mapping for listing
    await redis.hset(f"user:sessions:{user_id}", jti, json.dumps(session_info))

async def revoke_session(user_id: str, jti: str):
    # 블랙리스트에 추가하여 JWT 즉시 무효화
    await redis.hdel(f"user:sessions:{user_id}", jti)
    await redis.setex(f"blacklist:{jti}", 3600, "revoked")
```

## 🏁 Definition of Done (DoD)
- [ ] 모든 Acceptance Criteria를 충족하는가?
- [ ] 세션 취소 시 해당 토큰이 즉시 무효화되는지 실시간 테스트 완료?
- [ ] 로그아웃 이력이 `AUDIT_LOG` 에 정확히 기록되는가?

## 🚧 Dependencies & Blockers
- Depends on: `INFRA-002`, `AUTH-001`
- Blocks: 없음
