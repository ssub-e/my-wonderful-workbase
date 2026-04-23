---
name: Feature Task
about: SRS 기반의 구체적인 개발 태스크 명세
title: "[Engine] F1-003: 자동 추론(Inference) 및 예측치 적재 파이프라인"
labels: 'forecast, backend, pipeline, priority:critical, phase:2'
assignees: ''
type: task
tags: [task, misc]
created: 2026-04-22
---
## 🎯 Summary
- 기능명: [F1-003] 매일 도는 스케줄링 예측(Inference) 및 DB 영적재 워커
- 목적: 매일 아침 재고 수집(F2-W04)이 종료된 직후 파이프라인에 의해 실행되며, 전일자 훈련된(또는 로드된) 머신러닝 `.pkl` 모델을 올려 내일자 피처 벡처를 예측 함수로 주입한다. 반환된 예측수량과 신뢰도를 `DB-007`(`FORECAST` 테이블)에 적재한다.

## 🔗 References (Spec & Context)
- SRS 아키텍처: [`SRS-V1.0.md#§3.6`](raw/assets/SRS-V1.0.md) — Inference API / Workflow
- 데이터 연계: REQ-FUNC-004 (예측값 및 신뢰도 반환 저장)

## ✅ Task Breakdown (실행 계획)

### 1. 추론(Inference) Engine 서비스 층 구축
- [ ] `app/domain/forecast/engine/inferencer.py` 작성:
  ```python
  import joblib
  import pandas as pd
  import math
  from app.domain.forecast.engine.feature_builder import build_inference_vector

  class ModelInferencer:
      def __init__(self, tenant_id: str, sku: str):
          self.model_path = f"models/{tenant_id}_{sku}_lgb.pkl"
          self.model = joblib.load(self.model_path) # 파일 또는 S3에서 로드
          
      async def predict_next_day(self) -> dict:
          # 1. 내일치 환경(예보 날씨 등) 벡터화 (F1-001 재활용)
          input_df: pd.DataFrame = await build_inference_vector(target_date="tomorrow")
          
          # 2. 추론
          prediction_float = self.model.predict(input_df)[0]
          
          # 3. 비즈니스 로직(반올림, 음수 방지)
          final_qty = max(0, math.ceil(prediction_float))
          
          # TODO: DB-007 모델인 Forecast DTO와 맞추어 리턴. 신뢰도는 이전 학습 로그나 편차로 임시 설정.
          return {"predicted_qty": final_qty, "confidence_level": 0.85}
  ```

### 2. ETL 연계용 Worker 함수 (Scheduler 포트)
- [ ] `app/domain/forecast/workers/inference_worker.py` 생성.
  - 시스템에 물려있는 모든 테넌트-SKU 쌍을 순회하며 위 `predict_next_day` 를 호출하고 결과물을 `save_forecast_with_factors` (DB-007) 로 DB에 밀어 넣는 벌크 루프 함수 작성 (`run_daily_inference()`).

## 🧪 Acceptance Criteria (BDD/GWT)

**Scenario 1: 음수 예측치 방어 비즈니스 로직**
- Given: 비가 오고 트렌드가 바닥이라 LightGBM 트리가 `prediction_float = -3.5` (선형회귀 특성) 를 뱉음
- When: 값이 클레스를 통과함
- Then: `max(0, ...)` 비즈니스 안전망이 작동하여 DB에 절대로 마이너스 예측 출고량이 쌓이지 않고 합리적인 `0` 으로 적재된다.

**Scenario 2: 파일 로드 결함 (서킷 브레이커)**
- Given: 해당 SKU를 위한 `.pkl` 파일이 학습된 적 없거나 지워진 상태.
- When: 워커가 `ModelInferencer` 를 초기화함.
- Then: `FileNotFoundError` 발생 시, 해당 SKU 추론은 에러를 로깅(Audit)한 후, 예측을 강제로 중단하지 않고 단순히 `predicted_qty = 0, confidence = 0.0` 또는 이동평균 기반 Rule-based 임시 추론을 폴백해 담아 끊김없는 운영을 보장한다.

## ⚙️ Technical Constraints
- 추론 모델(`joblib.load`)은 IO 연산이 크므로, 클래스 초기화 시 한 번만 선별 로드하거나 FastAPI 의 `lru_cache` 를 이용해 메모리에 띄워두고 재사용하는 패턴(Global Serving)을 고민할 것.

## 🏁 Definition of Done (DoD)
- [ ] Vector Input을 받아들이는 Inferencer 래퍼가 작성?
- [ ] 소수점/음수 예측치를 보정(Ceil, Max 0)하는 비즈니스 룰 탑재?
- [ ] 예측 결과를 `DB-007` 레포지토리에 저장하는 Worker 로직 완성 유무?
