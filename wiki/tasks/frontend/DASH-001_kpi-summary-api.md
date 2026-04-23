---
name: Feature Task
about: SRS 기반의 구체적인 개발 태스크 명세
title: "[API] DASH-001: 대시보드 메인 KPI 요약 통계 API"
labels: 'api, frontend-integration, phase:3, priority:high'
assignees: ''
type: task
tags: [task, frontend]
created: 2026-04-22
---
## 🎯 Summary
- 기능명: [DASH-001] 메인 대시보드 상단 헤더용 KPI 데이터 제공 모듈
- 목적: REQ-FUNC-008 (대시보드 KPI 카드) 요구에 따라, 로그인한 테넌트 관리자가 화면을 켜자마자 보게 될 핵심 지표(전일 실제 주문량, 내일의 예측 출고량 전체 합산치, 평균 AI 신뢰도 등)를 한 번의 API 콜로 요약해 내려준다.

## 🔗 References (Spec & Context)
- 대시보드 요구사항: [`SRS-V1.0.md#§4.1.6`](file:///e:/workspace/SRS-from-PRD/SRS-V1.0.md) — REQ-FUNC-008
- 보안/격리: `DB-011` (Row-Level Isolation 상속형 Repository 적용 필수)

## ✅ Task Breakdown (실행 계획)

### 1. Pydantic DTO (응답 스펙) 설계
- [ ] `app/schemas/dashboard_schema.py`:
  ```python
  from pydantic import BaseModel
  from typing import Optional

  class DashboardKPISummary(BaseModel):
      tenant_id: str
      yesterday_actual_qty: int
      tomorrow_predicted_qty: int
      qty_variance_percent: float # (내일예측 - 어제실제) / 어제실제 * 100
      avg_confidence_level: float # 0.0 ~ 1.0
      last_sync_time: str
  ```

### 2. FastAPI Router 및 Service 구현
- [ ] `app/api/endpoints/dashboard.py`:
  ```python
  from fastapi import APIRouter, Depends
  from sqlalchemy.ext.asyncio import AsyncSession
  from app.core.database import get_session
  from app.api.dependencies.auth import get_current_tenant
  from app.services.dashboard_service import get_kpi_summary
  
  router = APIRouter()

  @router.get("/summary", response_model=DashboardKPISummary)
  async def read_kpi_summary(
      tenant_id: str = Depends(get_current_tenant),
      session: AsyncSession = Depends(get_session)
  ):
      """
      현재 로그인한 화주사의 전일 매출 합산과 익일 발주 예측 합산을 
      계산하여 KPI 카드용으로 반환한다.
      """
      return await get_kpi_summary(session, tenant_id)
  ```

### 3. Service 내부 조인 효율성 (캐싱)
- [ ] 메인화면 로딩 속도 방어를 위해, `get_kpi_summary` 내부에 `Redis` (INFRA-002) 조회 룰을 추가하여 5분 단위의 TTL(Time-To-Live) 기반 캐싱 맵을 두어 반복적인 `SUM()` 쿼리 부하를 차단한다.

## 🧪 Acceptance Criteria (BDD/GWT)

**Scenario 1: 증감 수식(Variance) 엣지 케이스 방어**
- Given: 고객이 어젯밤 막 가입하여 어제 매출(`yesterday_actual_qty`)이 0건인 상태
- When: 대시보드 API가 증감율 `qty_variance_percent` 를 계산하여 반환하려함
- Then: 분모가 0이 되어 `ZeroDivisionError`가 나지 않고, 안전망에 의해 `100.0` 이나 미리 약속한 Nullish 값(`None`)으로 치환되어 떨어진다.

**Scenario 2: 타 테넌트 침범 제어**
- Given: A화주사가 꼼수로 `GET /dashboard/summary?tenant_id=B` 를 파라미터로 날림
- When: Router에 연결된 `get_current_tenant` 의존성이 작동함
- Then: JWT 토큰을 뜯어 추출한 ID와 입력된 파라미터가 다름을 인지하거나, 아예 파라미터를 무시하고 무조건 JWT 기반의 토큰 ID로만 세션 조회를 강제하여 교차 노출을 막는다.

## ⚙️ Technical Constraints
- 대시보드의 N개 타일 중 하나이므로, 응답 시간(Latency)은 200ms 이내를 준수해야 한다 (SQL 집계 최적화 또는 Redis Cache 적극 활용).

## 🏁 Definition of Done (DoD)
- [ ] `DashboardKPISummary` 스펙을 준수하는 GET 엔드포인트가 구성되었나?
- [ ] `get_current_tenant` 를 통한 Security 의존성 주입이 적용되었는가?
