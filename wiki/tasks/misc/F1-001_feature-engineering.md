---
name: Feature Task
about: SRS 기반의 구체적인 개발 태스크 명세
title: "[Engine] F1-001: 머신러닝 학습용 Feature Engineering 결합 파이프라인"
labels: 'forecast, engine, data-science, priority:critical, phase:2'
assignees: ''
type: task
tags: [task, misc]
created: 2026-04-22
---
## 🎯 Summary
- 기능명: [F1-001] 예측 타겟(y)과 외부 피처(X)를 결합하는 엔지니어링 모듈
- 목적: 파편화되어 수집된 `ORDER`(매출/주문량, y값), `WEATHER_DATA`(기온/강수, X1), `TREND_DATA`(검색량, X2), 그리고 날짜 파장(주말/휴일, X3)을 하나의 Pandas(또는 Polars) DataFrame으로 조인하여 LightGBM 이 섭취할 수 있는 구조화된 학습/추론용 데이터 셋을 만든다.

## 🔗 References (Spec & Context)
- 모델 요구사항: [`SRS-V1.0.md#§4.1.1`](raw/assets/SRS-V1.0.md) — REQ-FUNC-001, 002 (날씨, 트렌드가 변수로 작동해야 함)
- 프로모션 변수: REQ-FUNC-017 (이벤트 여부 등 주문 테이블 기반 Meta 피처)
- 기술 스택: C-TEC-005 (머신러닝 전처리 파이프라인)

## ✅ Task Breakdown (실행 계획)

### 1. 피처 추출 및 조인 로직 구현
- [ ] `app/domain/forecast/engine/feature_builder.py` 모듈 작성:
  ```python
  import pandas as pd
  from typing import List
  from sqlalchemy.ext.asyncio import AsyncSession
  from app.adapters.persistence.order_repository import get_historical_orders_by_sku
  from app.adapters.persistence.weather_repository import get_weather_history
  
  async def build_training_dataset(session: AsyncSession, shop_id: str, sku: str) -> pd.DataFrame:
      # 1. 원천 데이터 Fetch
      orders = await get_historical_orders_by_sku(session, shop_id, sku) # 일별 합산된 출고량
      weather = await get_weather_history(session) # 해당 권역의 일별 날씨
      
      # 2. DataFrame 변환 (pydantic to dict)
      df_orders = pd.DataFrame([o.model_dump() for o in orders])
      df_weather = pd.DataFrame([w.model_dump() for w in weather])
      
      # 3. Time-based Join (일자 기준 Left Join)
      df = pd.merge(df_orders, df_weather, left_on='order_date', right_on='date', how='left')
      
      # 4. 파생 피처(Derived Features) 생성 알고리즘
      df['is_weekend'] = df['order_date'].dt.dayofweek >= 5
      df['temp_drop_5deg'] = df['temperature'].diff() <= -5.0 # 전일대비 5도 이상 급강하 여부
      df['rolling_3d_sales'] = df['total_qty'].rolling(window=3).mean() # 3일 이동평균선
      
      # 피처(X)와 라벨(y)이 하나의 배열로 정렬된 DataFrame 리턴
      # 결측치(NaN) 처리 루틴 포함 (ex: fillna(0) 또는 ffill)
      return df.dropna(subset=['total_qty'])
  ```

### 2. 향후 실시간 추론(Inference)용 추출기 병행 작성
- [ ] 내일(Target Date) 단 하루의 특성(예측된 내일 날씨, 최근 3일 이동평균값 등)만 추출하여 `[1 x N]` 형태의 벡터를 뱉는 `build_inference_vector` 함수 작성.

## 🧪 Acceptance Criteria (BDD/GWT)

**Scenario 1: 결측치(Null) 방어 및 Join 무결성**
- Given: 과거 1년치 주문 데이터가 존재하나, 중간에 기상청 서버 에러로 날씨 데이터가 빈 3일이 있음
- When: `build_training_dataset` 을 수행함
- Then: 반환되는 DataFrame은 주문 데이터(`total_qty`)의 Row 수를 온전히 보존(Left Join)하며 날씨가 비어있는 부분을 평균 또는 `0.0` 으로 Imputing 처리해 학습 런타임 오류(NaN Exception)를 방어한다.

**Scenario 2: 파생 피처 산출**
- Given: 12월 24일 금요일의 데이터가 들어옴
- When: 파생 피처 코드 라인을 통과함
- Then: `is_weekend` 변수가 True(1) 로 할당되어 주말 물량 폭등 모델링 가중치에 기여할 수 있는 준비가 된다.

## ⚙️ Technical Constraints
- 대용량 데이터(수백만 건) 조인 시 Pandas 의 병목 현상이 발생할 수 있으므로, 초기 단계부터 DB 레벨 질의(`GROUP BY`, `JOIN` 등)를 통해 최대한 집계된 상태(Aggregated)로 DB에서 꺼내오는 것을 원칙으로 한다. Pandas는 파생 피처 계산에만 사용.

## 🏁 Definition of Done (DoD)
- [ ] 주문(y), 날씨(X1), 프로모션플래그(X2) 가 하나로 합쳐진 Pandas DF 리턴 함수 작성?
- [ ] 누락값 처리(Imputation) 전략 코드 반영?
- [ ] 요일, 이동 평균 등 파생 변수(Feature) 생성 로직이 존재하는가?
