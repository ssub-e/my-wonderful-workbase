---
name: Feature Task
about: SRS 기반의 구체적인 개발 태스크 정세
title: "[API] ADMIN-005: 매뉴얼 데이터 수집/역산 동기화 패치 API"
labels: 'api, admin, phase:4, priority:low'
assignees: ''
type: task
tags: [task, admin]
created: 2026-04-22
---
## 🎯 Summary
- 기능명: [ADMIN-005] 데이터 수집, 역산 스케줄 수동 트리거(Manual Sync) API
- 목적: 기본적으로 데이터는 새벽/아침 스케줄러(APScheduler)에 의해 동작하나, 관리자가 새로 설정값을 바꾼 직후이거나 카페24 점검으로 아침에 누락본이 결측된 경우 "지금 당장 새로고침" 할 수 있는 강제 트리거 스위치를 제공한다.

## 🔗 References (Spec & Context)
- 기반 구조: `ETL-001` 의 Worker 개별 함수들 (ex: `run_order_collection_worker`)
- 연관 뷰: 프론트엔드의 "데이터 업데이트" 버튼

## ✅ Task Breakdown (실행 계획)

### 1. 워커 개별 실행 라우터 연결
- [ ] `app/api/endpoints/admin_sync.py`:
  ```python
  from fastapi import APIRouter, BackgroundTasks, Depends
  # 기존 만들어둔 파이프라인 워커 임포트
  from app.domain.integration.workers.order_worker import run_order_collection_worker
  from app.domain.workforce.services.calculator import calculate_workforce_demand

  @router.post("/sync/orders", status_code=202)
  async def trigger_manual_order_sync(
      background_tasks: BackgroundTasks,
      tenant_id: str = Depends(get_current_tenant)
  ):
      """ 
      동기 상태로 실행시 카페24 주문가져오는데 수 분이 걸릴 수 있으므로 
      무조건 BackgroundTasks 로 넘겨 202 응답.
      """
      # 특정 단일 테넌트만 긁어오도록 파라미터를 넘겨주어 Worker 실행
      background_tasks.add_task(run_order_collection_worker, target_tenant=tenant_id)
      return {"message": "주문 수집 백그라운드 작업이 지시되었습니다."}
  ```

### 2. 역산 데이터 리프레시 엔진 노출
- [ ] `POST /sync/workforce` API:
  - `ADMIN-003`에서 1인당 CAPA를 바꾼 사용자가 내일까지 기다리지 않고 "즉시 재계산"을 원할 때 호출용. (DB `WORKFORCE_PLAN` 레코드 해당 자 값 Update / Upsert 트리거)

## 🧪 Acceptance Criteria (BDD/GWT)

**Scenario 1: 동시 다발적 수동 갱신 버튼 클릭(Debouncing/Lock)**
- Given: 불안에 빠진 사용자가 "주문 동기화" 버튼을 10번 연속 누름.
- When: 이 API로 10번의 호출이 인입됨.
- Then: 이 API의 앞단 미들웨어나 Redis 락킹을 통해 `sync_lock_{tenant}` 가 잡혀있음을 판단하고 두번째 클릭부터는 `429 Too Many Requests (직전 작업이 아직 진행중입니다)` 를 내려보내 DB 커넥션 포화를 이겨낸다.

## ⚙️ Technical Constraints
- 전체 워커(`run_order_collection_worker` 등)는 기본으로 전 화주사를 다 돌도록 `F2-W03` 에서 짜여졌습니다. 따라서 워커 함수의 서명을 고쳐 `target_tenant=None` 을 허용하고, API 호출시에는 특정 테넌트만 배열에 잡히게 끔 코어 로직의 마이너한 리팩토링이 수반되어야 합니다.

## 🏁 Definition of Done (DoD)
- [ ] BackgroundTasks 를 이용한 빠른 202 Accepted 반환 구조 구현?
- [ ] DB/캐시 락 체계를 통해 과도한 연속 트리거를 방어하는가?
