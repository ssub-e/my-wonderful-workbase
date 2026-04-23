---
name: Feature Task
about: SRS 기반의 구체적인 개발 태스크 명세
title: "[Feature] VIEW-019: 시계열 차트 포인트 클릭 기반 '일별 상세 내역' 드릴다운(Drill-down) 모달"
labels: 'feature, frontend, ux, priority:medium'
assignees: ''
type: task
tags: [task, frontend]
created: 2026-04-22
---
## 🎯 Summary
- 기능명: [VIEW-019] 대시보드 차트 드릴다운(Drill-down) 인터랙션 UI
- 목적: REQ-FUNC-010 고도화. 사용자가 시계열 차트에서 특정 날짜의 튀는 지점(Outlier)을 발견했을 때, 해당 지점을 클릭하여 "왜 이 날만 물량이 많은가?" 에 대한 답변(SKU별 상세 비중, 센터별 분산 현황)을 레이어 팝업으로 즉각 확인하게 한다.

## 🔗 References (Spec & Context)
> 💡 AI Agent & Dev Note: 작업 시작 전 아래 문서를 반드시 먼저 Read/Evaluate 할 것.
- 기반 컴포넌트: `VIEW-006` (Area+Line 차트)
- 관련 API: `API-003` (드릴다운 통계 API - Batch 17 신설)

## ✅ Task Breakdown (실행 계획)
- [ ] Recharts `<AreaChart>` 혹은 `<LineChart>` 에 `onClick` 핸들러 바인딩 및 클릭된 데이터 포인트의 `date` 값 추출
- [ ] `Radix UI Dialog` (또는 기존 Tailwind Modal)를 활용한 상세 내역 팝업 컴포넌트 작성
- [ ] 선택된 날짜(`target_date`)를 파라미터로 하여 `API-003` 을 호출하는 React Query 훅 연결
- [ ] 모달 내부에 해당 일자의 'Top 5 SKU 기여도' 원형 차트(Pie Chart) 및 '센터별 적정 인원' 요약 테이블 렌더링
- [ ] 모달 하단에 '해당 일자 전체 리포트(PDF) 바로가기' 링크 배치

## 🧪 Acceptance Criteria (BDD/GWT)
Scenario 1: 차트 포인트 클릭 시 상세 정보 노출
- Given: 11월 12일 출고량이 평소보다 2배 높게 예측된 차트 화면
- When: 유저가 11월 12일의 데이터 포인트를 마우스로 클릭함
- Then: 화면 중앙에 모달이 뜨며 "11월 12일 분석 상세" 문구와 함께, 등산화 SKU의 비중이 70%라는 데이터 그리드가 로딩 스켈레톤 후에 안전하게 표출된다.

Scenario 2: 데이터 부재 시 예외 처리
- Given: 과거 데이터가 존재하지 않는 아주 먼 미래의 포인트를 클릭함
- When: 드릴다운 API가 빈 응답을 반환함
- Then: 모달 내부에 "해당 일자의 상세 분석 데이터가 아직 집계되지 않았습니다" 라는 안내 문구를 띄우고 빈 화면 크래시를 방지한다.

## ⚙️ Technical & Non-Functional Constraints
- 성능: 모달 오픈 시 애니메이션 지연 ≤ 200ms 유지. 데이터 로딩 중에는 `VIEW-015` 의 스켈레톤 UI 재사용
- UX: 모달이 떴을 때 배경 스크롤 방지 및 `ESC` 키로 닫기 지원 (Accessibility)
- 보안: 모달 레벨에서도 `tenant_id` 필터링이 유지되어 타사 상세 내역이 노출되지 않도록 전역 Context 확인

## 🏁 Definition of Done (DoD)
- [ ] 모든 Acceptance Criteria를 충족하는가?
- [ ] Recharts 포인트 클릭 시 정확한 날짜 스트링(YYYY-MM-DD)이 파라미터로 넘어가는가?
- [ ] 모달 내 차트와 테이블이 반응형(Responsive)으로 깨지지 않고 보이는가?

## 🚧 Dependencies & Blockers
- Depends on: `VIEW-006`, `API-003` (백엔드 데이터 공급)
- Blocks: 없음
