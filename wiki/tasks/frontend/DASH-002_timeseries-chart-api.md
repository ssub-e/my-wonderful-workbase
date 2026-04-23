---
name: Feature Task
about: SRS 기반의 구체적인 개발 태스크 명세
title: "[API] DASH-002: 시계열 차트 렌더링용 예측-실적 데이터 API"
labels: 'api, frontend-integration, phase:3, priority:critical'
assignees: ''
type: task
tags: [task, frontend]
created: 2026-04-22
---
## 🎯 Summary
- 기능명: [DASH-002] 프론트엔드 차트용 시계열(Time-series) 데이터 통합 API
- 목적: REQ-FUNC-009(과거-미래 연계 그래프)를 구현하기 위함. 하나의 점선/실선 복합 차트를 그리기 위해, 과거 N일치 실제 출고량(Order DB)과 미래 M일치 예측 출고량(Forecast DB)을 하나의 연속된 Array 로 가공하여 프론트엔드 도구(Recharts)가 파싱하기 쉬운 모양으로 제공한다.

## 🔗 References (Spec & Context)
- SRS 뷰 명세: [`SRS-V1.0.md#§4.1.6`](raw/assets/SRS-V1.0.md) — REQ-FUNC-009
- 기반 DB: `DB-003`(ORDER), `DB-007`(FORECAST)

## ✅ Task Breakdown (실행 계획)

### 1. Chart 지원용 DTO 작성
- [ ] `app/schemas/dashboard_schema.py` 내 추가:
  ```python
  from pydantic import BaseModel
  from typing import List, Optional

  class ChartDataPoint(BaseModel):
      date: str # "MM/DD" 포맷
      actual_qty: Optional[int] = None # 과거일 경우 값 존재, 미래일경우 Null
      predicted_qty: Optional[int] = None # 과거일경우 Null, 미래일경우 예측값 존재
      is_forecast: bool = False

  class ChartResponse(BaseModel):
      series: List[ChartDataPoint]
  ```

### 2. FastAPI 병합 라우터 구현
- [ ] `app/api/endpoints/dashboard.py` 하위 메서드 추가:
  ```python
  @router.get("/chart", response_model=ChartResponse)
  async def get_timeseries_chart(
      days_past: int = 14,
      days_future: int = 7,
      tenant_id: str = Depends(get_current_tenant),
      session: AsyncSession = Depends(get_session)
  ):
      """오늘을 기준으로 과거 14일, 미래 7일의 시계열 배열 반환"""
      # 1. DB에서 과거 구간 ORDER 합계 Fetch
      past_orders = await get_orders_grouped_by_date(session, tenant_id, days_past)
      
      # 2. DB에서 미래 구간 FORECAST 합계 Fetch
      future_forecasts = await get_forecasts_grouped_by_date(session, tenant_id, days_future)
      
      # 3. 빈 데이터(날짜 이빨 빠짐) 보정에 유의하여 List[ChartDataPoint] 조립 로직
      return generate_seamless_chart_list(past_orders, future_forecasts)
  ```

## 🧪 Acceptance Criteria (BDD/GWT)

**Scenario 1: 공백(이빨 빠진) 날짜 제로필(Zero-fill) 처리**
- Given: 과거 14일 치 조회를 요청했으나, DB 상 주문이 아예 없던 일자가 이틀 존재함
- When: `generate_seamless_chart_list`가 작동
- Then: DB 쿼리에서 누락된 날짜더라도 차트가 끊기지 않도록, 내부 달력 발생기(Calendar Generator)를 통해 해당 날짜를 채우고 `actual_qty=0` 으로 보정해 온전한 14칸짜리 배열을 무조건 리턴한다.

**Scenario 2: 과거와 미래가 맞닿는 '오늘(Today)' 오버랩 처리**
- Given: 프론트엔드 차트 라이브러리가 실선(과거)과 점선(미래)을 자연스럽게 잇기 위해 분기점 데이터가 필요함
- When: 어레이 조합 생성
- Then: 배열의 기준 인덱스인 '오늘' 데이터 노드에는 `actual_qty`(현재까지 접수량) 와 `predicted_qty`(오늘 모델이 예측했던 양) 이 둘 다 프로퍼티로 할당되어 존재해야 꺾은선 두 점이 부드럽게 이어질 수 있다.

## ⚙️ Technical Constraints
- SQL 쿼리 시, 개별 주문을 `sum()` 하는 `GROUP BY date` 형태의 집계질의(Aggregation Query)를 SQLAlchemy 계층에 명확히 선언해야 인메모리 포화 오류를 방지할 수 있다.

## 🏁 Definition of Done (DoD)
- [ ] `ChartDataPoint` 규격으로 맞춰진 `actual` 과 `predict` 분리 응답 구조가 완성되었나?
- [ ] Date Range 기준으로 값이 없는 날짜 시계열을 채워넣는(Zero-fill) 유틸리티가 작성됨?
