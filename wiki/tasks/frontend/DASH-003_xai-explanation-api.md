---
name: Feature Task
about: SRS 기반의 구체적인 개발 태스크 명세
title: "[API] DASH-003: XAI 해설 (SHAP + LLM 텍스트) 반환 API"
labels: 'api, xai, frontend-integration, phase:3, priority:high'
assignees: ''
type: task
tags: [task, frontend]
created: 2026-04-22
---
## 🎯 Summary
- 기능명: [DASH-003] AI 판단 해설(XAI) 전용 엔드포인트
- 목적: B2B 시스템의 신뢰도 확보(REQ-FUNC-005, 006)를 위한 핵심 뷰. 프론트엔드가 특정 날짜/특정 상품을 클릭했을 때, 왜 그런 수치가 나왔는지 모델의 **가중치 영향 표(Factors)**와 LLM이 번역한 **자연어 텍스트(Explanation)** 둘을 엮어서 전송한다.

## 🔗 References (Spec & Context)
- XAI 기능 요건: [`SRS-V1.0.md#§4.1.1`](file:///e:/workspace/SRS-from-PRD/SRS-V1.0.md)
- 기초 테이블: `DB-007` (`FORECAST` 단일 로우 내의 `explanation_text` 및 자식 `FORECAST_FACTOR` 활용)

## ✅ Task Breakdown (실행 계획)

### 1. XAI 응답 스펙 명세
- [ ] `app/schemas/xai_schema.py`:
  ```python
  from pydantic import BaseModel
  from typing import List

  class FactorItem(BaseModel):
      factor_name: str # ex: "강수량 증가", "휴일 효과"
      impact_value: float # ex: -12.5 (감소 요인), +8.0 (증가 요인)
      
  class XAIResponse(BaseModel):
      target_date: str
      predicted_qty: int
      confidence_level: float
      human_readable_explanation: str
      key_factors: List[FactorItem]
  ```

### 2. API Router 편제
- [ ] `app/api/endpoints/xai.py`
  ```python
  @router.get("/{sku}/explanation", response_model=XAIResponse)
  async def get_forecast_explanation(
      sku: str,
      target_date: str, # 조회 대상 일자
      tenant_id: str = Depends(get_current_tenant),
      session: AsyncSession = Depends(get_session)
  ):
      # 1. DB-007 로 선언된 레포지토리에서 대상 Forecast 와 조인된 Factors 를 인출 (Eager Loading)
      forecast_data = await get_forecast_with_factors(session, tenant_id, sku, target_date)
      
      if not forecast_data:
          raise HTTPException(status_code=404, detail="해당 날짜의 예측/XAI 데이터가 없습니다.")
          
      # 2. DTO 변환 및 리턴
      return XAIResponse(
          target_date=str(forecast_data.target_date),
          predicted_qty=int(forecast_data.predicted_qty),
          confidence_level=forecast_data.confidence_level,
          human_readable_explanation=forecast_data.explanation_text or "분석 데이터 생성 지연 중입니다.",
          key_factors=[FactorItem(factor_name=f.factor_type, impact_value=f.shap_value) 
                       for f in forecast_data.factors]
      )
  ```

## 🧪 Acceptance Criteria (BDD/GWT)

**Scenario 1: 조인 실패 없는 Eager Loading 구동**
- Given: 부모 `FORECAST` 레코드 및 이에 종속된 자식 `FORECAST_FACTOR` 3개가 달린 DB 객체 조립 건
- When: FastAPI 가 응답 직렬화(Serialization)를 수행하기 전
- Then: SQLAlchemy `selectinload` 혹은 Eager Loading이 걸려있어 `forecast_data.factors` 를 접근할 때 `LazyLoad` 런타임 에러(비동기 미세션 오류)가 나지 않고 깔끔하게 DTO 로 맵핑되는 것을 보장한다.

**Scenario 2: LLM 데이터 생성 지연 시 방어 조치**
- Given: 예측(Inference) 은 끝났으나 외부 LLM 의 지연으로 `explanation_text` 가 DB에 `NULL` 로 남아있는 찰나의 순간
- When: 클라이언트가 화면을 조작해 API를 터치
- Then: `404` 대신 응답을 허용하되, `explanation_text` 에는 백엔드에서 주입한 "분석 데이터 지연 중"이라는 안전한 문자열이 노출되어 UI 붕괴(Null 렌더)를 피할 수 있다.

## ⚙️ Technical Constraints
- `shap_value` 에 양/음수 기호가 살아서 그대로 프론트엔드에 전달되어야, 화면에서 빨간색(+) / 파란색(-) 막대 그래프를 적법하게 칠할 수 있다. 변형/가공하지 말고 넘긴다.

## 🏁 Definition of Done (DoD)
- [ ] `XAIResponse` DTO 구조 정립
- [ ] Parent-Children 레코드를 한 번에 가져와 DTO로 반환하는 API 핸들러 작성?
- [ ] Lazy Load 에러를 막기 위한 비동기 조인 쿼리문법 검수 완료?
