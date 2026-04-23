---
name: Feature Task
about: SRS 기반의 구체적인 개발 태스크 명세
title: "[Feature] ADMIN-008: 과금 대시보드 및 미결제 테넌트 일괄 제어 관리자 전용 뷰"
labels: 'feature, frontend, admin, priority:medium'
assignees: ''
type: task
tags: [task, admin]
created: 2026-04-22
---
## 🎯 Summary
- 기능명: [ADMIN-008] 슈퍼 어드민용 과금 관리 및 미결제 강제 통제 스크린
- 목적: 시스템 운영팀(Super Admin)이 전체 테넌트의 매출 현황, 결제 실패 건, 플랜 만료 예정 사를 한눈에 파악하고 필요 시 특정 테넌트의 권한을 수동으로 일시 정지(Suspend)하거나 플랜을 강제 조정(Override)할 수 있는 제어판을 제공한다.

## 🔗 References (Spec & Context)
> 💡 AI Agent & Dev Note: 작업 시작 전 아래 문서를 반드시 먼저 Read/Evaluate 할 것.
- SRS 문서: [`e:\workspace\SRS-from-PRD\SRS-V1.0.md#STK-06 시스템 관리자`](#)
- 상위 태스크: `SUPER-001` (테넌트 상태 확장)

## ✅ Task Breakdown (실행 계획)
- [ ] `ADMIN_BILLING_OVERVIEW` 통계 API 개발 (월간 총 매출 추이, 플랜별 유저 수)
- [ ] 미결제/연체 테넌트 리스트 조회 API (결제 시도 3회 실패 이상 타겟) 구현
- [ ] React 관리자 전용 Data Grid(`VIEW-013` 기반 확장): 테넌트명, 플랜, 정기결제액, 결제상태, 최근 결제일
- [ ] 강제 플랜 조정 모달 UI (Update Plan Tier & Status) 추가
- [ ] 특정 테넌트 대상 "결제 안내" 수동 알림톡 대량 발송(Batch SMS) 기능 연동

## 🧪 Acceptance Criteria (BDD/GWT)
Scenario 1: 결제 실패 테넌트 식별 및 수동 정지
- Given: 카드 한도 초과로 정기 결제가 1주일째 안 되고 있는 테넌트 A가 존재함
- When: 슈퍼 어드민이 '미결제 테넌트' 탭에 진입하여 해당 테넌트를 확인
- Then: 리스트에 빨간색으로 'Overdue' 경고가 표시되며, '상태 변경' 버튼을 클릭해 해당 테넌트를 수동으로 `SUSPENDED` 시켜 대시보드 접근을 완전히 차단할 수 있다.

Scenario 2: 프로모션 대상 테넌트 요금제 일괄 상향
- Given: 이벤트 기간 내 가입한 10개의 테넌트에게 'Pro' 플랜 1개월 무료 혜택 부여가 필요함
- When: 관리자가 다중 선택 후 '플랜 강제 변경' 을 클릭함
- Then: DB 내 10개 레코드가 일괄 업데이트되며, 각 테넌트 관리자에게 "무료 체험이 시작되었습니다" 알림톡이 즉시 발송된다.

## ⚙️ Technical & Non-Functional Constraints
- 성능: 전체 테넌트 리스트 조회 시 오프셋 페이징 적용 (p95 ≤ 500ms 지연 허용)
- 보안: 일반 테넌트 관리자가 이 API에 절대 접근할 수 없도록 `SUPER_ADMIN` 역할 검증 필수
- 안정성: 대량 업데이트 시 트랜잭션 보장 (모두 성공하거나 실패)

## 🏁 Definition of Done (DoD)
- [ ] 모든 Acceptance Criteria를 충족하는가?
- [ ] `/api/v1/super-admin/billing/*` 엔드포인트에 대한 접근 권한이 RBAC로 격리되었는가?
- [ ] 통계 차트(Recharts) 가시성이 확보되었는가?

## 🚧 Dependencies & Blockers
- Depends on: `BILLING-001`, `SUPER-002`
- Blocks: 없음
