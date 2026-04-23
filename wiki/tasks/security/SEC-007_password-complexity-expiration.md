---
name: Feature Task
about: SRS 기반의 구체적인 개발 태스크 명세
title: "[Feature] SEC-007: 비밀번호 복잡도 정책 엔포스먼트 및 만료(Expiration) 관리 로직"
labels: 'feature, backend, security, priority:high'
assignees: ''
type: task
tags: [task, security]
created: 2026-04-22
---
## 🎯 Summary
- 기능명: [SEC-007] 엔터프라이즈 보안을 위한 패스워드 생명주기 및 강도 강화
- 목적: 단순한 비밀번호 사용으로 인한 해킹 위험을 방지하기 위해 최소 복잡도(영문/숫자/특수문자 조합)를 강제하고, 90일 주기로 비밀번호 변경을 요청하며, 최근 사용했던 비밀번호로의 재사용을 금지하는 보안 정책을 시스템 레벨에서 구현한다.

## 🔗 References (Spec & Context)
> 💡 AI Agent & Dev Note: 작업 시작 전 아래 문서를 반드시 먼저 Read/Evaluate 할 것.
- 기본 인증: `AUTH-001` (JWT/Bcrypt)
- 요구사항: REQ-NF-015 (강력한 보안 표준 준수)

## ✅ Task Breakdown (실행 계획)
- [ ] **Password Validator** 강화:
  - 최소 8자 이상, 대소문자/숫자/특수문자 중 3종 이상 조합 체크 (Regex)
  - 유저 이메일이나 이름이 비밀번호에 포함되어 있는지 체크
- [ ] `USER` 테이블 확장: `password_updated_at` 필드 추가 및 데이터 마이그레이션
- [ ] **Password History** 시스템: 최근 사용한 비밀번호 해시 3개를 저장하는 전용 테이블 구축 및 변경 시 대조
- [ ] **Expiration Logic**: 
  - 로그인 시 `password_updated_at` 으로부터 90일 경과 여부 확인
  - 경과 시 "비밀번호를 변경해야 합니다" 라는 `PASSWORD_EXPIRED` 특수 에러 코드 반환
- [ ] 프론트엔드: 비밀번호 만료 에러 수신 시 '비밀번호 변경 전용 페이지'로 강제 랜딩 처리

## 🧪 Acceptance Criteria (BDD/GWT)
Scenario 1: 취약한 비밀번호 설정 거부
- Given: 신규 가입 또는 변경 화면
- When: "12345678" 또는 "password" 입력
- Then: 서버는 `422 Unprocessable Entity` 와 함께 정책 미준수 사유를 구체적으로 반환한다.

Scenario 2: 비밀번호 만료 및 의무 변경
- Given: 마지막 변경일이 100일 전인 유저
- When: 로그인을 시도함
- Then: 로그인은 승인되되, 토큰에 `password_expired: true` 클레임이 포함되거나 401 변형 에러를 통해 변경 화면으로 유도되어야 한다.

## ⚙️ Technical & Non-Functional Constraints
- 성능: 비밀번호 히스토리 대조 시 해시 연산(Bcrypt)이 여러 번 발생하므로 CPU 부하에 유의
- 보안: 만료되어도 즉시 접근 차단할지, 3일간의 유예 기간(`Grace Period`)을 줄지 여부는 테넌트별 설정 가능하도록 설계
- UX: 비밀번호 입력 시 우측에 '복잡도 점수(Strength Meter)' 시각적 표시

## 💻 Implementation Snippet (Complexity & Expire Check)
```python
import re
from datetime import datetime, timedelta

def validate_password_strength(password: str, user_email: str):
    if len(password) < 8:
        raise ValueError("Min length 8")
    if user_email.split('@')[0] in password:
        raise ValueError("Common words detected")
    # Regex for mixed types
    if not re.search(r"[A-Z]", password) or not re.search(r"\d", password):
        raise ValueError("Must contain upper case and digit")
    return True

async def check_password_expiry(user: User):
    if user.password_updated_at < datetime.utcnow() - timedelta(days=90):
        return True # Expired
    return False
```

## 🏁 Definition of Done (DoD)
- [ ] 모든 Acceptance Criteria를 충족하는가?
- [ ] `zxcvbn` 또는 유사 라이브러리를 통한 강도 측정 로직이 연동되었는가?
- [ ] 비밀번호 변경 시 `AUDIT_LOG` 에 'USER_PASSWORD_CHANGED' 가 기록되는가?

## 🚧 Dependencies & Blockers
- Depends on: `AUTH-001`
- Blocks: 없음
