---
name: Feature Task
about: SRS 기반의 구체적인 개발 태스크 명세
title: "[Feature] ANALYSIS-004: 예측치와 리드타임을 결합한 '품절 위험(Out-of-Stock) 및 적정 재고' 분석"
labels: 'feature, backend, analysis, priority:medium'
assignees: ''
type: task
tags: [task, data_analysis]
created: 2026-04-22
---
## 🎯 Summary
- 기능명: [ANALYSIS-004] 품절 방지를 위한 선제적 재고 경보 시스템
- 목적: 단순한 출고량 예측을 넘어, 현재 보유 재고량과 입고 리드타임(Lead Time)을 AI 예측치와 결합 분석한다. 재고가 소진될 것으로 예상되는 시점을 날짜 단위로 시뮬레이션하고, "지금 발주하지 않으면 3일 뒤 품절됩니다" 라는 강력한 경고와 함께 적정 발주 권장량을 제안한다.

## 🔗 References (Spec & Context)
> 💡 AI Agent & Dev Note: 작업 시작 전 아래 문서를 반드시 먼저 Read/Evaluate 할 것.
- 데이터 모델: `INVENTORY` (현재량), `FORECAST_DATA` (미래량), `PRODUCT.lead_time`
- 요구사항: REQ-FUNC-022 (재고 소진 임계치 알람)

## ✅ Task Breakdown (실행 계획)
- [ ] **Stockout Projection Engine** 구축:
  - `(현재 재고 + 입고 예정) - AI 일일 예측치` 를 매일 누적 계산하여 재고가 0이 되는 'Estimated Stockout Date' 산출
- [ ] **Safety Stock Calculator**: 변동성(Standard Deviation)을 고려한 안전 재고(Safety Stock) 권장 레벨 계산 로직 추가
- [ ] **Risk Status Mapping**:
  - `CRITICAL`: 리드타임 이내 소진 예상 (즉시 발주 필요)
  - `WARNING`: 7일 이내 소진 예상
  - `STABLE`: 14일 이상 여유
- [ ] **Alert Integration**: Critical 상태 전환 시 즉시 알림톡(`ADAPT-004`) 및 대시보드 팝업 전송
- [ ] **Procurement Suggestion API**: 품절을 막기 위해 오늘 시점에 필요한 최적 발주량(`Order Up-to Level`) 반환

## 🧪 Acceptance Criteria (BDD/GWT)
Scenario 1: 품절 위험 자동 감지
- Given: 현재 재고 100개, 하루 평균 20개 판매 예측, 입고 리드타임 5일
- When: 분석 엔진 구동
- Then: 5일 뒤 재고가 소진될 것으로 보이므로 리드타임과 정확히 겹친다. 상태는 'CRITICAL'로 표시되며 발주 권장량이 즉시 계산되어 노출되어야 한다.

Scenario 2: 안전 재고 반영 결과
- Given: 판매량이 들쭉날쭉한(변동성 높은) SKU A
- When: 적정 재고 분석 실행
- Then: 단순 평균치보다 높은 수준의 '안전 재고' 기준선이 차트상에 점선으로 표시되어야 한다.

## ⚙️ Technical & Non-Functional Constraints
- 정확성: 입고 예정 데이터(`INBOUND_PLAN`)가 있을 경우 이를 합산에 반드시 포함
- 성능: 전체 SKU에 대한 시뮬레이션은 부하가 크므로, 재고 변동이 발생한 SKU에 대해서만 증분 연산(Incremental Update) 수행
- UX: 시계열 차트(`VIEW-006`) 상에 재고 소진 예상 시점을 X표시로 시각화

## 💻 Implementation Snippet (Stockout Simulation Logic)
```python
def predict_stockout_date(current_stock, daily_forecasts, lead_time_days):
    remaining = current_stock
    for i, qty in enumerate(daily_forecasts):
        remaining -= qty
        if remaining <= 0:
            stockout_day_index = i
            is_risk = stockout_day_index <= lead_time_days
            return {
                "days_until_oos": stockout_day_index,
                "is_critical": is_risk,
                "suggested_order_qty": sum(daily_forecasts[:lead_time_days*2]) # 예시 로직
            }
    return {"status": "stable"}
```

## 🏁 Definition of Done (DoD)
- [ ] 모든 Acceptance Criteria를 충족하는가?
- [ ] `PRODUCT` 테이블에 리드타임(Lead Time) 컬럼이 존재하고 정상 참조되는가?
- [ ] 알림톡 전송 시 품절 예상 SKU 명칭이 정확히 포함되는가?

## 🚧 Dependencies & Blockers
- Depends on: `F1-003` (Inference), `DB-004` (Inventory)
- Blocks: 없음
