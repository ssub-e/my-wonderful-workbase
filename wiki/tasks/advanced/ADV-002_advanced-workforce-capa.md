---
name: Feature Task
about: SRS 기반의 구체적인 개발 태스크 명세
title: "[Advanced] ADV-002: 숙련도 분기를 포함한 고급형 알바 역산(Workforce) 계산기"
labels: 'backend, algorithm, extension, phase:2, priority:low'
assignees: ''
type: task
tags: [task, advanced]
created: 2026-04-22
---
## 🎯 Summary
- 기능명: [ADV-002] Phase 2: 주/야간 교대 및 숙련도 등급 기반 인력(CAPA) 산출 모듈
- 목적: `F3-001` 에서 구현한 1차원적인 나눗셈(`(예측물량 - 기본재고) / 1인당CAPA`)을 넘어, "고급 알바는 1.5배 속도를 내고, 야간조는 0.8배 속도를 낸다" 는 디테일한 수치를 반영하여 현장에 더 핏-인(Fit-in) 되는 인원 추천 엔진을 고도화한다.

## 🔗 References (Spec & Context)
- 확장 모듈 기반: `F3-001` (Workforce Calculator)
- 환경 변수: `VIEW-011` (어드민에 숙련도별 입력칸 추가 필요 - 추후)

## ✅ Task Breakdown (실행 계획)

### 1. 고급형 수학 공식 알고리즘 구성
- [ ] `app/domain/workforce/advanced_calculator.py`:
  ```python
  import math

  def calculate_advanced_workers(predicted_qty: int, params: dict) -> dict:
      """
      params: {
         "base_capa": 100,
         "night_shift_ratio": 0.3, # 30% 물량을 야간에 철야로 뺌
         "night_penalty": 0.85, # 야간 피로도에 따른 효율 하락
         "expert_ratio": 0.2, # 전체 알바중 20%는 고인물(고수련자)
         "expert_boost": 1.5  # 고수는 1.5배 일을 잘함
      }
      """
      # 1. 주/야간 물량 찢기
      day_qty = predicted_qty * (1 - params["night_shift_ratio"])
      night_qty = predicted_qty * params["night_shift_ratio"]
      
      # 2. 1인당 체감 실질 CAPA 산출 (가중 평균)
      # 평균인 = (고수 비율 * 1.5) + (초보 비율 * 1.0) 
      avg_efficiency = (params["expert_ratio"] * params["expert_boost"]) + ((1 - params["expert_ratio"]) * 1.0)
      
      real_day_capa = params["base_capa"] * avg_efficiency
      real_night_capa = (params["base_capa"] * avg_efficiency) * params["night_penalty"]
      
      # 3. 올림 처리하여 절대 인명수 반환
      return {
          "day_workers": math.ceil(day_qty / real_day_capa),
          "night_workers": math.ceil(night_qty / real_night_capa),
          "total": math.ceil(day_qty / real_day_capa) + math.ceil(night_qty / real_night_capa)
      }
  ```

### 2. 전략 패턴(Strategy Pattern) 오버라이드
- [ ] 시스템에서 테넌트별로 `use_advanced_calc = True` 인 회사에만 위 함수가 돌고, 나머지 일반 영세 쇼핑몰은 가벼운 `F3-001` 오리지널 함수가 돌게끔 분기 스위치를 만든다.

## 🧪 Acceptance Criteria (BDD/GWT)

**Scenario 1: 고급 가중치와 오리지널 계산기의 결괏값 편차 증명**
- Given: 내일 주문 예측이 똑같이 1,000 박스이고 베이스 CAPA가 똑같이 100 인 A사와 B사.
- When: A사는 `F3-001` 기본 계산기를 타고, B사는 "야간 비율 30%, 숙련공 비율 20%" 가 포함된 고급 모형을 탄다.
- Then: A사는 단순하게 `1000/100 = 10`명이 파견 통보되나, B사는 주간조 6명 + 야간조 4명(`night_penalty 감소치 반영`) = 나뉘어진 고급 인력 분할 값으로 제시되며 실무 호환성이 높다.

## ⚙️ Technical Constraints
- 소수점 연산 제어: 사람의 수는 무조건 자연수(Integer)여야 하므로, 5.1명이 필요하더라도 부족분 보충을 위해 `round()` 반올림이 아닌 `math.ceil()` 올림을 써서 방어적으로 넉넉히 인력을 부르도록 산수를 조정한다 (현장 결품 대란 방지 목적).

## 🏁 Definition of Done (DoD)
- [ ] 야간(Night) 페널티와 숙련공(Expert) 가중치가 포함된 수식 구현 유무?
- [ ] Output 인력이 무조건 자연수 올림 처리 및 주/야 딕셔너리로 분리 반환되는지 확인?
