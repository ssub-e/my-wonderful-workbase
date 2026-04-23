---
name: Feature Task
about: SRS 기반의 구체적인 개발 태스크 명세
title: "[Engine] F1-002: LightGBM 예측 모델 학습 및 튜닝 모듈"
labels: 'forecast, engine, machine-learning, priority:critical, phase:2'
assignees: ''
type: task
tags: [task, misc]
created: 2026-04-22
---
## 🎯 Summary
- 기능명: [F1-002] LightGBM 기반 수요 예측(Regression) 모델 학습 스크립트
- 목적: `F1-001`을 통해 전처리된 DataFrame을 바탕으로, SKU 별로 미래 출고량을 예측(Regression)하는 **LightGBM** 모델을 훈련시킨다. 모델 바이너리를 로컬 또는 S3에 저장/버저닝 한다. (C-TEC-005 규격)

## 🔗 References (Spec & Context)
- 핵심 분석 모델: [`SRS-V1.0.md#§4.1.1`](raw/assets/SRS-V1.0.md) — REQ-FUNC-004 (수요 예측 엔진 및 신뢰도 산출)
- 기술 제약: C-TEC-005 (scikit-learn 이나 DL 대신, 속도와 설명력이 입증된 LightGBM 프레임워크 한정 강제)

## ✅ Task Breakdown (실행 계획)

### 1. 학습 파이프라인 클래스 구축
- [ ] `app/domain/forecast/engine/model_trainer.py` 작성:
  ```python
  import lightgbm as lgb
  import joblib
  import os
  from sklearn.model_selection import train_test_split
  from sklearn.metrics import mean_squared_error, mean_absolute_error
  import pandas as pd

  class LightGBMTrainer:
      def __init__(self, tenant_id: str, sku: str):
          self.tenant_id = tenant_id
          self.sku = sku
          self.model_path = f"models/{tenant_id}_{sku}_lgb.pkl"
          self.params = {
              'objective': 'regression',
              'metric': 'rmse',
              'boosting_type': 'gbdt',
              'learning_rate': 0.05,
              'feature_fraction': 0.8,
              'verbose': -1
          }
          
      def train(self, df: pd.DataFrame, target_col: str = 'total_qty'):
          feature_cols = [c for c in df.columns if c not in [target_date, target_col]]
          X = df[feature_cols]
          y = df[target_col]
          
          # Time-series aware split (최신 20%를 테스트로)
          split_idx = int(len(X) * 0.8)
          X_train, X_valid = X[:split_idx], X[split_idx:]
          y_train, y_valid = y[:split_idx], y[split_idx:]
          
          train_data = lgb.Dataset(X_train, label=y_train)
          valid_data = lgb.Dataset(X_valid, label=y_valid, reference=train_data)
          
          model = lgb.train(
              self.params,
              train_data,
              num_boost_round=500,
              valid_sets=[valid_data],
              callbacks=[lgb.early_stopping(stopping_rounds=50)]
          )
          
          # Metrics 산출 (신뢰도 환산용)
          preds = model.predict(X_valid)
          rmse = mean_squared_error(y_valid, preds, squared=False)
          mae = mean_absolute_error(y_valid, preds)
          
          # 모델 바이너리 저장
          os.makedirs(os.path.dirname(self.model_path), exist_ok=True)
          joblib.dump(model, self.model_path)
          
          return {"rmse": rmse, "mae": mae, "model_path": self.model_path}
  ```

### 2. 신뢰도(Confidence Level) 스코어링 규칙 제정
- [ ] MAPE(Mean Absolute Percentage Error) 등을 활용 수치화. `1 - (MAE / 평균출고량)` 공식을 적용해 0~1 사이의 직관적인 Float(100% 기반) 통계를 `RMSE` 값과 별도로 산출. (대시보드 표시용)

## 🧪 Acceptance Criteria (BDD/GWT)

**Scenario 1: 조기 종료 (Early Stopping) 작동**
- Given: Overfitting(과적합)을 유발할 수 있는 충분히 많은 Epoch (num_boost_round=1000)
- When: `lgb.train` 이 검증셋(valid_data) 을 섭취하며 루프를 돎
- Then: 50 라운드 연속 진전(Validation Loss 감소)이 없으면, 학습이 즉시 조기 종료 커트되며 최적 피팅된 가중치 모델만이 `joblib.dump` 로 기록된다.

**Scenario 2: Time-series 특성 보존 (데이터 분할 검증)**
- Given: 1월부터 12월까지 시계열로 딕트된 DataFrame.
- When: Train / Valid 데이터가 분할 됨
- Then: `train_test_split`의 Random 분할을 끄고 순서를 지키거나 직접 Index(80%)를 슬라이싱 함으로써, "미래 데이터를 통해 과거를 훈련하는 (Data Leakage)" 치명적 실수가 차단된다.

## ⚙️ Technical Constraints
- CPU 바운드 오퍼레이션이므로 FastAPI 애플리케이션 루프를 막으면 안 됨. 이 워커는 BackgroundTasks 또는 별도 프로세스(`ProcessPoolExecutor`) 에서 비동기 호출(`run_in_threadpool`)로 래핑되어 실행되어야 한다.

## 🏁 Definition of Done (DoD)
- [ ] LghtGBM Regression 파라미터 세팅 및 Early-stopping 구현 완료?
- [ ] 시계열의 무결성(Data Leakage)을 피하는 Split 구조 생성 완료?
- [ ] 저장 가능한 바이너리 파일 `.pkl` 생성이 검증되었는가?
- [ ] 신뢰도 예측 점수 변환 식이 문서 혹은 코드에 자리 잡았나?
