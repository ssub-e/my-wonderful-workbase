---
name: Feature Task
about: SRS 기반의 구체적인 개발 태스크 명세
title: "[Advanced] ADV-001: 쇼핑몰 자체 할인/기획전(Promotion) 가중치 Feature 반영"
labels: 'backend, AI, extension, phase:2, priority:low'
assignees: ''
type: task
tags: [task, advanced]
created: 2026-04-22
---
## 🎯 Summary
- 기능명: [ADV-001] Phase 2: 쇼핑몰 내부 할인 및 기획전 변수(Promotion Weight) 결합 머신러닝 모듈
- 목적: 기상청 정보나 요일 트렌드(`F1` 기본엔진)만으로는 갑작스럽게 화주사가 자체 세일을 걸었을 때 폭주하는 물량을 맞출 수 없다. 따라서 해당 일자에 '자체 프로모션'이 있었는지 여부(Boolean 혹은 할인율 Float)를 AI의 추가 Feature 로 집어넣도록 파이프라인을 고도화한다.

## 🔗 References (Spec & Context)
- 확장 대상 모듈: `F1-001` (Feature Engineering Pandas 데이터 프레임)
- 참조 모델 제약: REQ-FN-012 (도메인 특화 가중치)

## ✅ Task Breakdown (실행 계획)

### 1. 과거 프로모션 이력 메타데이터 스키마 확장
- [ ] `DB-007` `FORECAST_FACTOR` 인근에, `PROMOTION_HISTORY` 라는 날짜(Date) 기반의 기획전 기간 기록용 작은 테이블 하나를 신설하거나 메타로 뺀다. (예: 11.11 빼빼로데이 세일 기간 = True)

### 2. Feature Engineering 결합체 후처리
- [ ] `app/ai/feature_engineer.py`:
  ```python
  import pandas as pd

  def append_promotion_features(df: pd.DataFrame, tenant_id: str) -> pd.DataFrame:
      # 1. DB에서 해당 테넌트에 걸렸던 과거 기획전 날짜와 진행 여부(has_promotion) 긁어옴
      promos = get_promotions_from_db(tenant_id) # DataFrame 형태
      
      # 2. Left 조합으로 붙여줌
      merged = pd.merge(df, promos, on='date', how='left')
      
      # 3. 빈 곳은 일반 평일(False/0)로 덤프
      merged['has_promotion'] = merged['has_promotion'].fillna(0).astype(int)
      merged['discount_rate'] = merged['discount_rate'].fillna(0.0)
      
      return merged
  ```

### 3. LightGBM 트리 재생성
- [ ] `F1-002` 재호출: 새로 붙인 `has_promotion` 컬럼이 SHAP 분석에 반영되도록 트리를 재학습 시킨다. 이 요인이 포함되면 그날 예측값이 크게(Spike) 치솟는 것을 확인할 수 있다.

## 🧪 Acceptance Criteria (BDD/GWT)

**Scenario 1: SHAP에 기획전 요소 등장 검증**
- Given: 평소 100건 나가던 모델링. 이틀 뒤에 `has_promotion=1` (기획전 진행) 세팅을 ADMIN 에서 걸어둠
- When: 새벽 1배치 `F1-003` 일일 추론 워커가 내일 분량(Inference)을 연산함
- Then: 이틀 뒤 예측 물량이 100건 -> 450건 으로 튀면서, XAI (`F1-004`) 해석 API에 `["기획전 진행 여부", impact_value: +350.0]` 이라는 강력한 양수 팩터가 잡혀 프론트엔드 대시보드에 노출된다.

## ⚙️ Technical Constraints
- 과거 학습 데이터에도 프로모션 이력(Y/N)이 세팅되어야 모델이 그 차이를 학습할 수 있다. 화주사가 SaaS 초기 가입시 "과거 1년치 세일했던 날좀 입력해달라"는 온보딩 UI가 필요하다는 한계가 있으므로, 우선은 하드코딩 혹은 CSV 업로드 형태의 심플한 데이터 적재로 타협한다.

## 🏁 Definition of Done (DoD)
- [ ] Pandas Merge 를 이용한 `has_promotion` Feature 추가 함수 구성?
- [ ] 결측치(NaN)일 경우 평일(0)로 채우는 방어 제어구문(`fillna`) 확보?
