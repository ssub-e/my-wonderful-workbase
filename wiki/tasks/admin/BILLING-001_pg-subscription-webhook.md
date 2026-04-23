---
name: Feature Task
about: SRS 기반의 구체적인 개발 태스크 명세
title: "[Feature] BILLING-001: PG(PortOne) 연동 정기 결제 웹훅 및 빌링키(Billing Key) 저장소"
labels: 'feature, backend, billing, priority:high'
assignees: ''
type: task
tags: [task, admin]
created: 2026-04-22
---
## 🎯 Summary
- 기능명: [BILLING-001] 외부 PG사 연동을 통한 정기 결제(Subscription) 자동화 엔진
- 목적: B2B SaaS로서 매월 정기적인 수익 모델을 확보하기 위해, 고객의 카드 정보를 빌링키 형태로 안전하게 암호화 보관하고 PortOne(구 아임포트) 웹훅을 통해 결제 성공/실패 여부를 테넌트 상태에 즉각 반영한다.

## 🔗 References (Spec & Context)
> 💡 AI Agent & Dev Note: 작업 시작 전 아래 문서를 반드시 먼저 Read/Evaluate 할 것.
- SRS 문서: [`e:\workspace\SRS-from-PRD\SRS-V1.0.md#6.2 Entity & Data Model`](#) (TENANT.plan_tier 참조)
- 외부 API: PortOne 정기결제(Subscription) REST API 명세서

## ✅ Task Breakdown (실행 계획)
- [ ] 결제 이력 저장용 `PAYMENT_HISTORY` 테이블 스키마 설계 (id, tenant_id, amount, status, imp_uid, merchant_uid)
- [ ] PortOne 가맹점 식별코드 및 REST API Key 환경변수(SEC-003 연계) 세팅
- [ ] 고객사 카드 등록(최초 1회) 시 발급받은 `customer_uid` (빌링키) 암호화 저장 로직 구현
- [ ] 정기 결제 결과 수신용 웹훅 엔드포인트 (`POST /api/v1/billing/webhook`) 구현
- [ ] 결제 실패 시 재시도 큐잉 및 `TENANT.status` 를 `DUE_EXPIRED` 로 변경하는 상태머신 로직

## 🧪 Acceptance Criteria (BDD/GWT)
Scenario 1: 정기 결제 웹훅 성공 시 테넌트 상태 유지
- Given: Pro 플랜을 이용 중인 테넌트A의 결제일 도래
- When: PortOne에서 결제 성공 웹훅(`status=paid`)이 도달함
- Then: `PAYMENT_HISTORY`에 이력이 남고, `TENANT.plan_tier`가 Pro로 유지되며 `next_billing_date`가 한 달 뒤로 자동 연장된다.

Scenario 2: 결제 실패(한도 초과 등) 시 테넌트 기능 정제
- Given: Basic 플랜 이용 유저의 등록 카드가 잔액 부족으로 결제 실패
- When: PortOne 웹훅(`status=failed`)이 수신됨
- Then: 해당 테넌트 관리자에게 "결제 실패" 알림톡(`F3-W03` 연계)이 발송되고, 테넌트 상태가 즉시 `SUSPENDED`로 변경되어 대시보드 접근이 차단된다.

## ⚙️ Technical & Non-Functional Constraints
- 성능: 웹훅 처리 응답시간 p95 ≤ 300ms (단순 로깅 후 비동기 처리)
- 안정성: PG사 장애 시 유저의 서비스 이용에 지장이 없도록 웹훅 유실 시 수동 재검증(Manual Reconciliation) 배치 모듈 마련
- 보안: 실제 카드 번호는 절대 서버에 저장하지 않으며 PG사가 발급한 `customer_uid`만 AES-256으로 암호화하여 취급

## 🏁 Definition of Done (DoD)
- [ ] 모든 Acceptance Criteria를 충족하는가?
- [ ] API 명세서에 웹훅 수신 규격이 최신화되었는가?
- [ ] 결제 실패 시나리오에 대한 알림 발송 연동이 완료되었는가?
- [ ] SQLModel 기반의 결제 이력 모델 및 마이그레이션 파일이 생성되었는가?

## 🚧 Dependencies & Blockers
- Depends on: `SUPER-001` (Tenant 상태 확장), `DB-001`
- Blocks: `BILLING-002` (기능 제한 미들웨어)
