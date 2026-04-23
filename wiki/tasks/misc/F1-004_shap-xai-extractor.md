---
name: Feature Task
about: SRS 기반의 구체적인 개발 태스크 명세
title: "[Engine] F1-004: 예측 원인(XAI) 기여도 추출 브릿지 모듈 (SHAP)"
labels: 'forecast, xai, priority:high, phase:2'
assignees: ''
type: task
tags: [task, misc]
created: 2026-04-22
---
## 🎯 Summary
- 기능명: [F1-004] SHAP 패키지를 이용한 요인(Factor) 기여도 추출 브릿지
- 목적: REQ-FUNC-005(XAI 설명 가능 인공지능)를 충족한다. LightGBM이 뱉은 단순한 정수(예: "내일 500개 나감") 뒤에 숨은 피처들의 상대적 기여도(예: 날씨가 추워서 +50건 기여)를 SHAP 로켓 알고리즘으로 분해하여 `FORECAST_FACTOR` (DB-007) 에 적재한다.

## 🔗 References (Spec & Context)
- 설명 데이터 보관: [`SRS-V1.0.md#§6.2`](raw/assets/SRS-V1.0.md) — FORECAST_FACTOR 자식 테이블
- 요구사항: REQ-FUNC-005 (SHAP 원인 추적 저장)
- 기술스택: C-TEC-006 (`shap` 라이브러리 사용 강제)

## ✅ Task Breakdown (실행 계획)

### 1. SHAP TreeExplainer 클래스 구현
- [ ] `app/domain/forecast/engine/xai_explainer.py` 작성:
  ```python
  import shap
  import pandas as pd
  from typing import List, Dict

  class ShapExplainer:
      def __init__(self, lightgbm_model):
          # LightGBM 트리에 최적화된 Explainer
          self.explainer = shap.TreeExplainer(lightgbm_model)

      def get_top_factors(self, input_df: pd.DataFrame, top_n: int = 3) -> List[Dict]:
          """
          추론을 위해 사용된 오늘치(1줄) Vector 를 던지면, 
          해당 예측값을 만든 가장 크리티컬한 Feature N개를 추출 반환
          """
          shap_values = self.explainer.shap_values(input_df)
          
          # DataFrame 구조로 매핑
          feature_names = input_df.columns
          # 단건 추론이므로 스칼라 1차원 배열 추출
          vals = shap_values[0] 
          
          # List of dict 구성
          factors = [{"name": name, "shap_val": float(val)} for name, val in zip(feature_names, vals)]
          
          # 절대값 크기(영향도) 순으로 정렬하여 상위 추출
          sorted_factors = sorted(factors, key=lambda x: abs(x["shap_val"]), reverse=True)
          
          return sorted_factors[:top_n]
  ```

### 2. Inferencer 파이프라인(F1-003)과 결합 연결
- [ ] 기존 추론 파이프라인에서 예측 후 즉각 이 모듈을 호출하여 반환 값을 `ForecastFactor` DTO 시퀀스로 감싸서 `save_forecast_with_factors` 로 DB에 동시에 때려 넣도록 결합.

## 🧪 Acceptance Criteria (BDD/GWT)

**Scenario 1: SHAP TreeExplainer 상위 요건 필터링 정상**
- Given: 어떠한 예측 결과와 피처 20개가 담긴 벡터 어레이
- When: `get_top_factors(df, top_n=3)` 을 호출
- Then: `[{"name": "temperature", "shap_val": -12.4}, {"name": "is_weekend", "shap_val": +5.2}, {"name": "search_volume", "shap_val": +2.1}]` 형태의 Dictionary 3개짜리 리스트가 반환되며, 불필요한 배경 변수(0에 수렴하는 쓰레기값)는 걸러져 DB 스토리지 낭비를 이겨낸다.

**Scenario 2: 모델 독립성(Dependency Injection)**
- Given: 파이썬 테스트 코드를 수행
- When: 더미 `lightgbm_model`을 Explainer의 생성자로 밀어 넣음
- Then: 하드코딩된 모델 Load 없이 순수하게 Explainer가 작동하여 단위 테스트 커버리지를 100% 충족시킬 수 있다.

## ⚙️ Technical Constraints
- `shap_values()` 연산의 비용이 상당히 비쌉니다(CPU-intensive). 반드시 **추론 벡터 건 단위(Row by Row)** 로만 단발 실행되도록 유의하여 서빙 시 서버 응답 블로킹 장애를 피해야 합니다.

## 🏁 Definition of Done (DoD)
- [ ] `shap.TreeExplainer` 구동 코드 스니펫 구현 반영?
- [ ] 양/음수를 아우르는 절대값(Abs) 정렬(Top 3) 추출 알고리즘 반영됨?
- [ ] 부동 소수점을 넘기지 않기 위해 Float 형변환이나 직렬화가 완료되었는가?
