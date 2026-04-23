---
name: Feature Task
about: SRS 기반의 구체적인 개발 태스크 명세
title: "[Feature] ANALYSIS-002: 화주사 이탈 징후 감지 및 '미접속/데이터 단절' 주간 경고 쿼리"
labels: 'feature, backend, analysis, priority:low'
assignees: ''
type: task
tags: [task, data_analysis]
created: 2026-04-22
---
## 🎯 Summary
- 기능명: [ANALYSIS-002] 테넌트 Retention 방어를 위한 이탈 징후 감지 배치 로직
- 목적: B2B 운영팀이 고객사의 이탈을 사전에 방지하기 위해, "최근 7일간 대시보드에 한 번도 접속하지 않았거나", "쇼핑몰 연동이 끊긴 채 방치되고 있는" 화주사 리스트를 추출하여 운영팀 슬랙(`ADAPT-006` 연계)으로 보고한다.

## 🔗 References (Spec & Context)
> 💡 AI Agent & Dev Note: 작업 시작 전 아래 문서를 반드시 먼저 Read/Evaluate 할 것.
- 가이드라인: SaaS Churn Rate 방어 전략 (Retention Analysis)
- 데이터 모델: `AUDIT_LOG`, `SHOP.status`

## ✅ Task Breakdown (실행 계획)
- [ ] `app/workers/retention_analyzer.py`: 주간 배치(Weekly Job) 스케줄 등록
- [ ] `Churn Detection SQL` 작성:
  - 접속 단절: `AUDIT_LOG` 에서 특정 `tenant_id` 의 최근 활동이 1주일 이상 부재한 경우
  - 연동 단절: `SHOP.status` 가 `disconnected` 가 된 지 48시간이 경과한 경우
  - 리포트 미채택: 생성된 예측 대비 PDF 다운로드 비율이 현저히 낮은 경우
- [ ] 추출된 리스트를 `Slack Webhook` 포맷으로 가공 (테넌트명, 단절 기간, 담당자 연락처 포함)
- [ ] 본사 운영팀 슬랙 채널 `#alert-customer-success` 로 자동 송출

## 🧪 Acceptance Criteria (BDD/GWT)
Scenario 1: 장기 미접속 테넌트 탐지 및 보고
- Given: 화주사 'A컴퍼니' 가 바쁜 시즌이 끝나고 10일간 대시보드에 로그인을 안 함
- When: 매주 월요일 오전 9시 리텐션 배치 잡이 실행됨
- Then: 슬랙에 "[경고] 이탈 위험 테넌트 발생: A컴퍼니 (최근 접속 10일 전)" 라는 메시지가 담당 MD 연락처와 함께 전송된다.

Scenario 2: 연동 실패 방치 건 탐지
- Given: 화주사 'B유통' 의 카페24 토큰이 만료되어 데이터 수집 워커가 2일째 에러를 뿜는 중
- When: 배치 잡 실행
- Then: "연동 데이터 단절 48시간 경과: B유통" 이라는 제목으로 즉각 리포팅되어 운영팀이 수동 조치를 취할 수 있게 돕는다.

## ⚙️ Technical & Non-Functional Constraints
- 성능: 전체 테넌트 스캔 시 인덱스(`AUDIT_LOG.created_at`) 활용으로 DB 부하 최소화
- 운영: 테스트 환경(Staging) 에서는 실제 슬랙 발송 대신 로그만 출력하도록 환경변수 분리
- 보안: 외부에 노출되는 슬랙 메시지에는 민감한 화주사 주문 데이터는 제외하고 상태값만 전송

## 🏁 Definition of Done (DoD)
- [ ] 모든 Acceptance Criteria를 충족하는가?
- [ ] 슬랙 메시지에 포함된 담당자 연락처 정보가 `ADMIN-001` 데이터와 일치하는가?
- [ ] 배치 잡이 정해진 요일과 시간에 정확히 트리거되는 것을 확인하였는가?

## 🚧 Dependencies & Blockers
- Depends on: `AUDIT-001`, `ADAPT-006` (Slack), `INFRA-003` (스케줄러)
- Blocks: 없음
