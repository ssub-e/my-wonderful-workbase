---
name: Feature Task
about: SRS 기반의 구체적인 개발 태스크 명세
title: "[Advanced] ADV-003: 자동 수집 워커(Worker) 서킷 브레이커 및 데이터 Fallback 전략"
labels: 'backend, worker, reliability, phase:2, priority:high'
assignees: ''
type: task
tags: [task, advanced]
created: 2026-04-22
---
## 🎯 Summary
- 기능명: [ADV-003] 통신망 마비 시 '유동적 데이터 복구(Fallback)' 로직 가동
- 목적: REQ-NF-015. 기상청 API 장애나 쇼핑몰 호스팅사의 대규모 점검 시, 해당 일자의 데이터가 `0건` 으로 DB에 박혀 멀쩡한 AI 수요예측 그래프 라인 전체가 붕괴(왜곡)되는 현상을 방어한다. 수집 3회 실패(서킷 브레이커 오픈 상태)시 최후의 수단으로 `전주 동일 요일` 의 데이터를 복사해 채워넣는 기믹을 추가한다.

## 🔗 References (Spec & Context)
- 타겟 레이어: `ETL-002` (기존의 단순 Retry/Queue 보강본)
- 종속 기능: `F2-W03` (주문 수집 워커)

## ✅ Task Breakdown (실행 계획)

### 1. Fallback Strategy 엔진
- [ ] `app/workers/fallback.py`:
  ```python
  from datetime import datetime, timedelta
  from sqlalchemy.orm import Session
  from app.domain.order.repository import OrderRepository
  from app.core.logger import logger

  def execute_fallback_for_orders(session: Session, target_date: datetime, tenant_id: str):
      """
      가망이 없을 때(Circuit Opened) 호출되어 빈 공간을 대체 데이터로 메꾼다.
      """
      repo = OrderRepository(session, current_tenant_id=tenant_id)
      
      # 지난주(7일 전) 동일 요일의 데이터 쿼리
      last_week_date = target_date - timedelta(days=7)
      past_data = repo.get_orders_by_date(last_week_date.date())
      
      if past_data:
          # 대체 복제 로직 (데이터 왜곡 방지를 위해 별도의 fallback_flag 를 true로 매핑하여 저장한다)
          for item in past_data:
             cloned = item.copy()
             cloned.id = None 
             cloned.collection_date = target_date.date()
             cloned.is_fallback_data = True  # AI 엔진이 참고할 메타데이터
             session.add(cloned)
          session.commit()
          logger.warning(f"[{tenant_id}] 크롤링 완전 실패. {last_week_date} 데이터를 Fallback 복제 완료.")
      else:
          logger.error(f"[{tenant_id}] Fallback 데이터마저 존재하지 않음 (치명적 결측).")
  ```

### 2. 데코레이터 통합 체인 구상
- [ ] `ETL-002` 에서 만든 `with_etl_retry` 데코레이터의 `except` 블록 최하단에서 `raise e` 로 Sentry만 날리고 끝내던 것을 개조하여, 에러 핸들링 직전에 특정 복구 함수(Fallback Callback)를 주사기처럼 주입해 실행되게 파라미터를 파싱한다.

## 🧪 Acceptance Criteria (BDD/GWT)

**Scenario 1: `is_fallback_data` 방어적 로깅 증명**
- Given: 외부망이 막혀 워커가 뻗고, 로직에 의해 7일 전 1,500건의 주문 내역이 오늘 날짜로 감쪽같이 이관(복제) 됨.
- When: 대시보드 API(`DASH-002`) 나 관리자가 해당 날짜의 데이터를 조회.
- Then: 데이터베이스 상에 `is_fallback_data = True` 라는 마커가 찍혀 있으므로, 프론트엔드가 차트를 그릴 때 이 부분만 "⚠ 타사 API 오류로 인한 추정치(지난주 기준)" 라는 경고 툴팁을 띄워 고객의 신뢰를 유지할 수 있다.

## ⚙️ Technical Constraints
- 서킷 브레이커가 터졌다고 무턱대고 0을 박아넣어 머신러닝의 시계열 `Sequence` 를 끊어버리면 다음날 LightGBM 예스/노 분기 트리가 걷잡을 수 없이 오염됨을 인지하고 AI 개발자는 이런 마커가 찍힌 로우(`Row`)를 어떻게 보정(Interpolation)시킬지 협의해야 한다.

## 🏁 Definition of Done (DoD)
- [ ] 7일 전 데이터를 긁어와 통째로 날짜만 갈아 복사하는 Fallback 스니펫 작성부?
- [ ] `is_fallback_data` boolean 플래그 컬럼 Schema 확장 포함인가?
