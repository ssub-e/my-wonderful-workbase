---
name: Feature Task
about: SRS 기반의 구체적인 개발 태스크 명세
title: "[Feature] DQ-001: 데이터 위생(Data Hygiene) 이상치 감지 및 정제 제안 워커"
labels: 'feature, backend, ai, logic, priority:medium'
assignees: ''
type: task
tags: [task, data_analysis]
created: 2026-04-22
---
## 🎯 Summary
- 기능명: [DQ-001] 데이터 정합성 보호를 위한 이상치(Outlier) 자동 탐지기
- 목적: 외부 API(카페24 등)로부터 수집된 원시 데이터 중, 비정상적인 전산 오류(예: 재고가 음수로 찍힘, 주문량이 평소보다 100배 많은 오타성 입력)를 예측 전 단계에서 걸러내어, AI 엔진의 오염(Data Poisoning)을 막고 유저에게 "데이터 확인 필요" 알림을 제공한다.

## 🔗 References (Spec & Context)
> 💡 AI Agent & Dev Note: 작업 시작 전 아래 문서를 반드시 먼저 Read/Evaluate 할 것.
- 관련 워커: `ETL-001` (파이프라인)
- 분석 라이브러리: `Scikit-learn (IsolationForest)` 또는 `Z-score` 통계 모델

## ✅ Task Breakdown (실행 계획)
- [ ] `app/logic/data_quality.py`: 통계 기반 이상치 탐지 유틸리티 클래스 작성
- [ ] `ORDER` 및 `INVENTORY` 수집 직후 실행되는 `DataHygieneWorker` 신설
- [ ] 탐지 로직 1: Z-score 기반 주문량 폭증(Spike) 감지 (최근 30일 평균 대비 3σ 초과)
- [ ] 탐지 로직 2: 논리적 모순 감지 (예: 판매량 > (기초재고 + 입고량))
- [ ] 탐지된 이상 데이터를 `DATA_ANOMALY` 테이블에 기록하고 관련 테넌트 MD에게 긴급 알림(`ADAPT-004`) 발송

## 🧪 Acceptance Criteria (BDD/GWT)
Scenario 1: 비정상 주문 폭증 감지
- Given: 평소 하루 평균 주문이 100건인 화주사
- When: 시스템 오작동으로 단일 주문에 수량 99,999개가 찍혀서 수집됨
- Then: `DQ-001` 워커가 이를 이상치로 감지하고, 해당 데이터를 'Pending Approval' 처리하며 예측 엔진 입력에서 일시 제외시킨다.

Scenario 2: 재고 음수값 감지
- Given: 카페24 API 응답 중 특정 상품의 재고가 `-50` 으로 수신됨
- When: 데이터 유효성 검사 수행
- Then: "재고 논리 오류 감지" 이벤트가 생성되고, 대시보드 상단에 "재고 데이터 불일치. 파트너사 전산 확인 요망" 경고가 뜬다.

## ⚙️ Technical & Non-Functional Constraints
- 성능: 대량 수집 시 전체 스캔 대신 미니 배치(Mini-batch) 처리 (수집당 ≤ 2초)
- 신뢰성: 단순 프로모션(폭증)과 실제 데이터 오류를 구분하기 위해 `is_promotion` 플래그가 있는 경우 임계값 완화
- 투명성: 왜 이상치로 판단했는지에 대한 근거(예: "30일 평균 대비 12배 초과")를 함께 저장

## 💻 Implementation Snippet (Anomaly Detection Idea)
```python
import numpy as np

def detect_order_spikes(daily_orders: List[float], threshold: float = 3.5):
    """
    Modified Z-score를 사용한 주문 폭증 감지
    """
    median = np.median(daily_orders)
    mad = np.median([abs(x - median) for x in daily_orders])
    
    if mad == 0: return [False] * len(daily_orders)
    
    modified_z_scores = [0.6745 * (x - median) / mad for x in daily_orders]
    return [abs(score) > threshold for score in modified_z_scores]

# Usage in Worker
is_anomaly = detect_order_spikes(recent_history + [new_order_qty])[-1]
if is_anomaly:
    await trigger_data_hygiene_alert(tenant_id, new_order_id)
```

## 🏁 Definition of Done (DoD)
- [ ] 모든 Acceptance Criteria를 충족하는가?
- [ ] 이상치 탐지 로직에 대한 Mock 데이터 테스트 케이스(Spike, Zero, Negative)가 통과되었는가?
- [ ] 탐지된 오류를 MD가 "수용" 혹은 "무시" 할 수 있는 상태 처리 API가 설계되었는가?

## 🚧 Dependencies & Blockers
- Depends on: `ETL-001`, `DB-003`
- Blocks: `DASH-010` (데이터 위생 대시보드)
