---
name: Feature Task
about: SRS 기반의 구체적인 개발 태스크 명세
title: "[Feature] SEC-004: 민감 설정 변경 시 이메일 2FA(OTP) 인증 보안 절차"
labels: 'feature, backend, security, priority:high'
assignees: ''
type: task
tags: [task, security]
created: 2026-04-22
---
## 🎯 Summary
- 기능명: [SEC-004] 관리자용 핵심 설정 변경 보호를 위한 이메일 2차 인증(2FA)
- 목적: REQ-NF-017 강화. 쇼핑몰 API 키(`ADMIN-002`), 테넌트 비밀번호 수정, 결제 수단 변경 등 보안상 치명적인 작업을 수행할 때, 로그인 세션 외에 계정 이메일로 발급된 6자리 핀(PIN) 번호를 입력하게 하여 계정 탈취(Account Takeover)에 의한 2차 피해를 방지한다.

## 🔗 References (Spec & Context)
> 💡 AI Agent & Dev Note: 작업 시작 전 아래 문서를 반드시 먼저 Read/Evaluate 할 것.
- 관련 태스크: `ADAPT-004` (알림/SMS 어댑터를 이메일로 확장하거나 SendGrid 등 연동)
- 보안 정책: 기업용 SaaS 보안 컴플라이언스 가이드

## ✅ Task Breakdown (실행 계획)
- [ ] `VERIFICATION_TOKEN` 신설: id, user_id, token_hash, type(2FA_SETTINGS), expires_at, is_used
- [ ] Redis 기반의 단기 유효(TTL 5분) OTP 저장소 연동 (`INFRA-002` 활용)
- [ ] 이메일 발송 워커: SendGrid 또는 SMTP를 통한 6자리 보안 코드 전송 로직
- [ ] 인증 프로세스 API: 
  - `POST /api/v1/auth/2fa/request` : 인증 코드 발송 요청
  - `POST /api/v1/auth/2fa/verify` : 코드 검증 및 임시 승인 토큰 발급
- [ ] 민감 설정 API 미들웨어: 유효한 2FA 승인 토큰이 헤더에 포함되었는지 강제 검증

## 🧪 Acceptance Criteria (BDD/GWT)
Scenario 1: API 키 변경 시도 시 2FA 요구
- Given: MD가 카페24 연동 키를 수정하려고 함
- When: 수정한 값을 '저장' 버튼 클릭
- Then: 즉시 저장되지 않고 "이메일로 전송된 보안 코드를 입력하세요" 라는 모달이 뜸. 이메일함에는 6자리 숫자가 도착함.

Scenario 2: 잘못된 코드 입력 시 차단
- Given: 2FA 입력 창이 활성화된 상태
- When: 유저가 유효하지 않거나 만료된(5분 경과) 코드를 3회 이상 입력함
- Then: 해당 인증 요청은 폐기되며, "보안 코드가 일치하지 않습니다" 메시지와 함께 작업이 반려됨.

## ⚙️ Technical & Non-Functional Constraints
- 보안: 2FA 코드는 평문으로 저장하지 않고 SHA-256 등으로 해싱하여 저장
- UX: 2FA 요청 후 1분간 재요청 금지(Rate Limit) 적용하여 메일 서버 부하 방지
- 가용성: 이메일 서버 장애 시 관리 콘솔 등 우회 경로 확보(Super Admin 전용)

## 💻 Implementation Snippet (Verification Logic Idea)
```python
import secrets
import redis

def generate_otp(user_id: str, r_client: redis.Redis):
    """6자리 숫자 OTP 생성 및 Redis 저장 (5분 만료)"""
    otp = str(secrets.randbelow(900000) + 100000)
    # 보안을 위해 OTP 자체를 키로 쓰지 않고 user_id를 키로 사용
    r_client.setex(f"2fa:{user_id}", 300, otp)
    return otp

async def verify_otp(user_id: str, input_otp: str, r_client: redis.Redis):
    """OTP 검증"""
    stored_otp = r_client.get(f"2fa:{user_id}")
    if stored_otp and stored_otp.decode() == input_otp:
        r_client.delete(f"2fa:{user_id}")
        return True
    return False
```

## 🏁 Definition of Done (DoD)
- [ ] 모든 Acceptance Criteria를 충족하는가?
- [ ] `ADMIN-002` (API 키 저장) 기능에 2FA 미들웨어가 성공적으로 적용되었는가?
- [ ] 이메일 템플릿에 기업 브랜딩 및 보안 경고 문구가 포함되었는가?

## 🚧 Dependencies & Blockers
- Depends on: `INFRA-002` (Redis), `ADAPT-004` (Email/SMS Interface)
- Blocks: 없음
