---
name: Feature Task
about: SRS 기반의 구체적인 개발 태스크 명세
title: "[Feature] DASH-009: 실시간 인건비 절감액 게이지(Gauge) 및 목표 대비 달성률(OKR) 대시보드"
labels: 'feature, frontend, ux, priority:medium'
assignees: ''
type: task
tags: [task, frontend]
created: 2026-04-22
---
## 🎯 Summary
- 기능명: [DASH-009] 누적 인건비 절감액 시각화 및 목표 달성률 게이지 바
- 목적: REQ-NF-035~039 증명. 단순 수치 제공을 넘어 "우리 시스템 도입 후 이번 달에 총 얼마를 아꼈는가?" 와 "목표 대비 몇 %나 인력 오버 부킹을 방어했나?" 를 게이지 차트로 보여주어 고객사가 SaaS의 가치를 즉각 체감(Moment of Aha)하게 한다.

## 🔗 References (Spec & Context)
> 💡 AI Agent & Dev Note: 작업 시작 전 아래 문서를 반드시 먼저 Read/Evaluate 할 것.
- SRS 문서: [`e:\workspace\SRS-from-PRD\SRS-V1.0.md#4.2.9 경쟁 대안 대비 벤치마크 목표`](#)
- 수식 기준: `(기존 직감 방식 일 평균 손실 80만) - (SaaS 도입 후 실제 잉여 손실액)`

## ✅ Task Breakdown (실행 계획)
- [ ] `Recharts` 혹은 `D3` 기반의 반원형 게이지(Gauge / Speedometer) 컴포넌트 구현
- [ ] `TENANT` 테이블 혹은 별도 설정값에서 관리하는 '이번 달 절감 목표액' 데이터 바인딩
- [ ] 백엔드로부터 누적 절감액 총계를 가져오는 `useQuery` 훅 연결
- [ ] 목표 도달 시(예: 100% 초과) 축하 애니메이션(Confetti 효과 등) 트리거 로직
- [ ] 해당 게이지 하단에 "현재까지 총 [N]명의 오버 부킹을 막았습니다" 라는 직관적인 카피 문구 배치

## 🧪 Acceptance Criteria (BDD/GWT)
Scenario 1: 절감액 실시간 렌더링
- Given: 현재까지 누적 절감액이 520만 원, 이번 달 목표가 1,000만 원인 상태
- When: 대시보드 메인 페이지에 진입함
- Then: 게이지 바의 바늘이 약 52% 지점을 가리키며, 중앙에 "520,000원 절감 중" 이라는 텍스트가 굵게 표시된다.

Scenario 2: 목표 초과 달성 시 시각적 강조
- Given: 운용 효율이 극대화되어 절감액이 목표(1,000만 원)를 넘어 1,200만 원에 도달함
- When: 화면 로드
- Then: 게이지 바가 가득 차고 색상이 'Success Green' 으로 변경되며 상단에 "목표 달성!" 뱃지가 활성화된다.

## ⚙️ Technical & Non-Functional Constraints
- 성능: 애니메이션이 부드럽게 돌아가도록 최적화 (60fps 유지)
- 신뢰성: 절감액 계산 로직이 실제 `WORKFORCE_PLAN` 데이터와 1원 단위까지 일치해야 함 (데이터 투명성)
- UX: 대시보드 최상단에 배치하여 로그인 즉시 보이도록 구성 (Leading Metric 우선순위)

## 🏁 Definition of Done (DoD)
- [ ] 모든 Acceptance Criteria를 충족하는가?
- [ ] 게이지 바의 바늘 애니메이션이 버벅임 없이 동작하는가?
- [ ] 수치가 변경될 때 (Refetch 시) 카운트업(Count Up) 효과가 적용되었는가?

## 🚧 Dependencies & Blockers
- Depends on: `VIEW-001`, `DASH-001`
- Blocks: 없음
