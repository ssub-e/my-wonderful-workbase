---
name: Feature Task
about: SRS 기반의 구체적인 개발 태스크 명세
title: "[Feature] BILLING-002: 구독 플랜별 기능 접근 제어 미들웨어 (Feature Gating)"
labels: 'feature, backend, security, billing, priority:high'
assignees: ''
type: task
tags: [task, admin]
created: 2026-04-22
---
## 🎯 Summary
- 기능명: [BILLING-002] 테넌트 구독 등급 기반 API/기능 차단막 (Feature Gating)
- 목적: 무료/베이직 사용자가 고비용 리소스(XAI, 1년 대량 이송 등)를 남용하는 것을 막고, 결제 만료 시 모든 읽기/쓰기 권한을 즉각 회수하여 SaaS 수익성을 보호한다.

## 🔗 References (Spec & Context)
> 💡 AI Agent & Dev Note: 작업 시작 전 아래 문서를 반드시 먼저 Read/Evaluate 할 것.
- SRS 문서: [`e:\workspace\SRS-from-PRD\SRS-V1.0.md#1.2.1 In-Scope`](#)
- 관련 의존성: `SUPER-004` (무료 회원 차단 로직 확장형)

## ✅ Task Breakdown (실행 계획)
- [ ] `app/core/billing/gate.py`: 구독 등급별 권한 매트릭스(Enum) 정의
- [ ] FastAPI `Depends` 시그니처를 활용한 전역 가드 함수 구현 (`check_plan_permission`)
- [ ] PDF 생성 및 AI 추론 엔드포인트에 `Plan: Pro` 전용 데코레이터 적용
- [ ] 결제 실패/정지(Suspended) 테넌트 대상 API 요청 시 402(Payment Required) 혹은 403 에러 반환 로직
- [ ] 프론트엔드(`VIEW-001`)에서 권한 없는 버튼을 비활성화(Gray-out)하기 위한 `permissions` API 연동

## 🧪 Acceptance Criteria (BDD/GWT)
Scenario 1: 무료 유저의 제한 기능 접근 시도
- Given: 'Free' 요금제를 사용하는 테넌트 계정
- When: 고사양 AI 해설 리포트 생성(`F1-W07`) API를 호출함
- Then: 서버는 "Pro 플랜 전용 기능입니다" 라는 메시지와 함께 `402 Payment Required` 를 반환한다.

Scenario 2: 유예 기간(Grace Period) 내 접근 허용
- Given: 결제가 어제 실패하여 'Payment Failed' 상태이지만, 시스템 설정상 유유기간이 3일로 지정됨
- When: 대시보드에 진입함
- Then: 접근은 허용하되 상단에 "결제 수단 확인이 필요합니다. 2일 후 서비스가 중지됩니다" 라는 경고 배너를 노출한다.

## ⚙️ Technical & Non-Functional Constraints
- 성능: 미들웨어 오버헤드 최소화 (Redis 캐시 `INFRA-002` 를 통해 테넌트 상태를 매번 DB 조회하지 않고 1분 캐싱 유지)
- 안정성: 에러율 ≤ 0.1% 유지
- 보안: 테넌트 ID 위변조 방지를 위해 JWT 토큰 내 `plan_tier` 클레임을 포함시키거나 요청 시마다 검증

## 🏁 Definition of Done (DoD)
- [ ] 모든 Acceptance Criteria를 충족하는가?
- [ ] 'Free', 'Pro', 'Enterprise' 각 등급별 접근 가능 기능 목록이 코드로 확정되었는가?
- [ ] 결제 유도(Checkout) 페이지로의 리다이렉션 로직이 백엔드 단에서 준비되었는가?

## 🚧 Dependencies & Blockers
- Depends on: `BILLING-001`, `AUTH-002`
- Blocks: 없음
