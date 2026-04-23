---
name: Feature Task
about: SRS 기반의 구체적인 개발 태스크 명세
title: "[Feature] ANALYSIS-001: Amplitude North Star Metrics 행동 로그 추적기"
labels: 'feature, frontend, analysis, priority:medium'
assignees: ''
type: task
tags: [task, data_analysis]
created: 2026-04-22
---
## 🎯 Summary
- 기능명: [ANALYSIS-001] 북극성 지표(1-Pass 통과율 등) 수집을 위한 Amplitude SDK 연동
- 목적: REQ-NF-030~032 지표를 측정하기 위해, 유저가 PDF를 생성할 때 값을 수정했는지, 혹은 역산된 인원수를 그대로 채택했는지 등의 '의사결정 데이터'를 익명화하여 추적하고 제품의 ROI를 수치로 증명한다.

## 🔗 References (Spec & Context)
> 💡 AI Agent & Dev Note: 작업 시작 전 아래 문서를 반드시 먼저 Read/Evaluate 할 것.
- SRS 문서: [`e:\workspace\SRS-from-PRD\SRS-V1.0.md#4.2.7 비즈니스 KPI`](#)
- 타켓 지표: `1-Pass 통과율`, `발주량 무수정 채택 비율`

## ✅ Task Breakdown (실행 계획)
- [ ] `@amplitude/analytics-browser` 라이브러리 설치 및 초기화 모듈 작성
- [ ] 유저 로그인 시 `setUserId` 를 통한 테넌트-유저 식별자 매핑 (개인정보는 해싱 처리)
- [ ] `REPORT_GENERATED` 이벤트 트래킹: `is_edited`, `forecast_id`, `template_id` 속성 포함
- [ ] `WORKFORCE_CONFIRMED` 이벤트 트래킹: `diff_count` (시스템 권장치 vs 실제 확정치 차이) 속성 포함
- [ ] Amplitude 대시보드 상에서 `North Star Metric` 깔때기(Funnel) 차트 구성 보조

## 🧪 Acceptance Criteria (BDD/GWT)
Scenario 1: PDF 리포트 생성 및 수정 여부 기록
- Given: 대시보드에서 예측 수치를 5건 수정하고 '리포트 생성' 버튼을 클릭함
- When: PDF 생성 프로세스가 시작됨
- Then: Amplitude로 `REPORT_GENERATED` 이벤트가 쏘아지며, 커스텀 속성으로 `is_edited: true` 가 정확히 기록된다.

Scenario 2: 테넌트별 그룹 속성 부여
- Given: 특정 테넌트(화주사 A) 소속 유저가 활동 중
- When: 모든 이벤트가 발생함
- Then: 이벤트 메타데이터에 `tenant_id` 와 `plan_tier` 속성이 공통으로 박혀, 추후 요금제별 행동 패턴 분석이 가능해진다.

## ⚙️ Technical & Non-Functional Constraints
- 성능: 이벤트 전송 시 메인 스레드 블로킹 방어 (비동기 큐잉 활용)
- 보안: PII(개인식별정보) 노출 절대 금지. 유저 ID는 고유 UUID 혹은 단방향 해시값만 사용
- 신뢰성: 오프라인 상태에서 발생한 이벤트는 `localStorage` 에 임시 보관 후 온라인 복구 시 일괄 전송(Batching)

## 🏁 Definition of Done (DoD)
- [ ] 모든 Acceptance Criteria를 충족하는가?
- [ ] Amplitude 실시간 디버거(Debugger)를 통해 이벤트가 누락 없이 수집됨을 확인하였는가?
- [ ] `is_edited` 플래그의 정합성(수정 시에만 True)이 단위 테스트로 검증되었는가?

## 🚧 Dependencies & Blockers
- Depends on: `VIEW-001`, `F1-W07` (리포트 생성 버튼)
- Blocks: `KPI-001` (지표 분석 인프라)
