---
name: Feature Task
about: SRS 기반의 구체적인 개발 태스크 명세
title: "[Feature] ANALYSIS-003: 예측 정확도 심층 분석을 위한 '세그먼트별 코호트(Cohort) 리포트'"
labels: 'feature, backend, analysis, priority:medium'
assignees: ''
type: task
tags: [task, data_analysis]
created: 2026-04-22
---
## 🎯 Summary
- 기능명: [ANALYSIS-003] SKU 그룹 및 유입 채널별 예측 정확도 코호트 분석
- 목적: 전체 평균 예측 정확도(WAPE 등)만으로는 알 수 없는 심층적 인사이트를 제공한다. 예를 들어 '신규 런칭 상품군'의 첫 달 정확도 변화, 혹은 'S급 베스트셀러'의 요일별 오차 추이를 코호트(집단)별로 묶어서 분석함으로써, AI 엔진의 장점과 약점을 파악하고 비즈니스 의사결정을 돕는다.

## 🔗 References (Spec & Context)
> 💡 AI Agent & Dev Note: 작업 시작 전 아래 문서를 반드시 먼저 Read/Evaluate 할 것.
- 통계 지표: WAPE, MAPE, Bias (예측 편향성)
- 관련 태스크: `DASH-002`, `DASH-010`

## ✅ Task Breakdown (실행 계획)
- [ ] **Cohort Aggregator** 구축:
  - SKU를 가동 시간/판매량/카테고리 별로 그룹화(Grouping) 하는 엔진 개발
  - 시간축(Time-axis)에 따른 정확도 변동 추이 집계 로직
- [ ] **Accuracy Heatmap UI**: X축(시간), Y축(세그먼트)을 기준으로 정확도를 색상으로 표현하는 히트맵 컴포넌트 구현
- [ ] **Bias Analysis**: AI가 특정 상품군에 대해 '과소 예측(Under-forecast)' 하는지 '과다 예측(Over-forecast)' 하는지 편향성 지수 산출
- [ ] **Insight Generator**: Gemini LLM(`ADAPT-005`) 연동을 통해 "A 브랜드 제품군은 월요일에 유독 오차가 큽니다" 와 같은 문장형 요약 생성
- [ ] 엔드포인트: `GET /api/v1/analysis/accuracy/segments`

## 🧪 Acceptance Criteria (BDD/GWT)
Scenario 1: 세그먼트별 정확도 비교
- Given: 1,000개의 SKU가 분류됨 (A: 고회전, B: 저회전)
- When: 코호트 리포트 화면 진입
- Then: 히트맵을 통해 고회전 상품군은 정확도가 95% 이상으로 일정하나, 저회전 상품군은 특정 요일에 70%대로 떨어진다는 패턴을 시각적으로 확인할 수 있어야 함.

Scenario 2: 예측 편향성(Bias) 탐지
- Given: 최근 4주간 계속해서 10% 정도 부족하게 예측된 센터 주문 데이터
- When: 분석 엔진 구동
- Then: Bias 지수가 음수(-)로 깊게 나타나며, 시스템은 "현재 인력 운용이 과소 평가되어 실전에서 인원 부족이 예상됩니다" 라는 경고를 노출해야 함.

## ⚙️ Technical & Non-Functional Constraints
- 성능: 복잡한 그룹화 연산이 필요하므로 `Pandas/Polars`를 활용한 메모리 내 고속 연산 수행
- UX: 히트맵의 각 셀을 클릭하면 해당 코호트에 속한 SKU 리스트로 드릴다운(`VIEW-019`) 연계
- 정확성: 실적이 아직 발생하지 않은 날짜는 정확도 계산 대상에서 제외 처리

## 💻 Implementation Snippet (Cohort Accuracy Logic)
```python
import pandas as pd

def calculate_cohort_wape(df: pd.DataFrame):
    """
    세그먼트별 WAPE(Weighted Absolute Percentage Error) 계산
    """
    total_actual = df.groupby('segment')['actual_qty'].sum()
    total_abs_error = df.groupby('segment')['abs_error'].sum()
    
    wape = (total_abs_error / total_actual) * 100
    return wape.to_dict()

# Example return: {"A_Brand": 4.5, "B_Brand": 12.8}
```

## 🏁 Definition of Done (DoD)
- [ ] 모든 Acceptance Criteria를 충족하는가?
- [ ] 대시보드 내 리포팅 속도가 2초 이내로 유지되는가?
- [ ] LLM이 생성한 통계 해석 문장이 도메인 지식에 비추어 타당한가?

## 🚧 Dependencies & Blockers
- Depends on: `F1-001`, `ADAPT-005`
- Blocks: 없음
