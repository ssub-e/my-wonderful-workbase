---
name: Feature Task
about: SRS 기반의 구체적인 개발 태스크 명세
title: "[Feature] ETL-003: 누락 및 데이터 오염 복구를 위한 '역사적 데이터 재동기화(Re-sync)' 툴"
labels: 'feature, backend, etl, priority:medium'
assignees: ''
type: task
tags: [task, backend_core]
created: 2026-04-22
---
## 🎯 Summary
- 기능명: [ETL-003] 과거 유실 데이터 복구 및 재연산 유틸리티
- 목적: 운영 중 외부 API 장애나 시스템 오류로 인해 특정 날짜의 주문 데이터가 수집되지 않았거나, 통계값이 오염되었을 경우 이를 수동 또는 자동으로 재수집(Re-fetch)하고 머신러닝 연산까지 다시 수행(Re-run)할 수 있는 관리자용 복구 툴을 제공한다.

## 🔗 References (Spec & Context)
> 💡 AI Agent & Dev Note: 작업 시작 전 아래 문서를 반드시 먼저 Read/Evaluate 할 것.
- 인프라: `ETL-001` (DAG), `F1-W01`~`F2-W04` (워커군)
- 관련 태스크: `ADMIN-005` (수동 갱신 API)

## ✅ Task Breakdown (실행 계획)
- [ ] **Gap Detection Logic**: DB상의 일자별 주문 건수 통계를 조회하여 데이터가 급감하거나 누락된 '공백 구간' 자동 탐지 모듈
- [ ] **Targeted Re-sync API**: 
  - 특정 테넌트, 특정 타겟일자(`start_date` ~ `end_date`)를 파라미터로 받아 해당 기간의 수집 워커를 강제 재가동
- [ ] **Cascade Update Trigger**: 
  - 주문 데이터가 복구된 후, 연관된 시계열 통계 및 AI 예측(`F1-003`)까지 순차적으로 다시 구동시키는 워크플로우 제어
- [ ] **Conflict Resolution**: 기존에 일부 존재하던 중복 데이터와 복구 데이터 간의 충돌 방지(`UPSERT` 로직 강화)
- [ ] **Progress Monitoring**: 대형 테넌트의 수개월 치 복구 작업 시 진행률(%)을 슈퍼 어드민 뷰에 실시간 표출

## 🧪 Acceptance Criteria (BDD/GWT)
Scenario 1: 특정 일자 데이터 공백 복구
- Given: 4월 15일 카페24 API 점검으로 데이터가 누락된 테넌트
- When: 관리자가 '4월 15일 재동기화' 버튼을 클릭함
- Then: 시스템은 해당 날짜의 API를 다시 훑어 유실된 주문을 채워넣고, 4월 16일 이후의 예측치를 새 데이터 기반으로 자동 재계산해야 한다.

Scenario 2: 대규모 마이그레이션 중단 및 재개
- Given: 1년 치 데이터를 복구 중인 상황
- When: 네트워크 오류로 작업이 도중에 끊김
- Then: 시스템은 어디까지 복구했는지 체크포인트(`ETL_SYNC_LOG`)를 기록하여, 재개 시 처음부터가 아닌 끊긴 시점부터 다시 시작해야 한다.

## ⚙️ Technical & Non-Functional Constraints
- 성능: 복구 작업은 운영 서버의 실시간 수집 업무에 지장을 주지 않도록 낮은 우선순위(Low Priority) 큐에서 처리
- 정합성: 재수집 시 기존에 MD가 수동으로 수정한 데이터(`NOTE-001`)가 덮어씌워지지 않도록 보호 정책 수립
- 로그: 모든 재동기화 수행 내역은 `AUDIT_LOG` 에 'ETL_DATA_RESYNC' 로 남겨야 함

## 💻 Implementation Snippet (Re-sync Task Trigger Idea)
```python
async def trigger_resync_workflow(tenant_id: str, start_date: date, end_date: date):
    # 1. 수집 워커 재가동
    sync_job = await run_worker_task("fetch_orders", tenant_id, start_date, end_date)
    
    # 2. 완료 대기 후 AI 연산 트리거 (Cascade)
    if sync_job.status == "SUCCESS":
        await run_worker_task("recalculate_forecast", tenant_id, start_date, end_date)
        return {"status": "Complete", "affected_rows": sync_job.rows}
```

## 🏁 Definition of Done (DoD)
- [ ] 모든 Acceptance Criteria를 충족하는가?
- [ ] 복구 도중 중복 데이터(Primary Key Conflict) 없이 `UPSERT`가 정상 동작하는가?
- [ ] 복구 직후 대시보드 차트가 새 데이터로 갱신되는 것을 확인했는가?

## 🚧 Dependencies & Blockers
- Depends on: `ETL-001`, `F1-003`
- Blocks: 없음
