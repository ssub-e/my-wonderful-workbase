---
name: Feature Task
about: SRS 기반의 구체적인 개발 태스크 명세
title: "[Engine] F3-001: 예측 출고량 기반 창고 인원(Workforce) 역산 모듈"
labels: 'backend, algorithm, phase:1, priority:medium'
assignees: ''
type: task
tags: [task, misc]
created: 2026-04-22
---
## 🎯 Summary
- 기능명: [F3-001] 예측 수량 ➡ 필요 적정 아르바이트생 수 역산 Rule-based 모듈
- 목적: 전체 복잡한 F1(예측 엔진) 파이프라인의 최종 목적이자 고객의 원초적 니즈 (Jobs-to-be-done)를 해결하는 기능(REQ-FUNC-019). 예측된 모든 내일치 출고 박스 수를 합산하고, 물류 센터별 1인당 적정 처리 수준(CAPA) 으로 나누어, 당장 내일 호출해야 할 적정 인원수(N명)를 도출해 낸다 (`DB-009` 투입).

## 🔗 References (Spec & Context)
- SRS 요구사항: [`SRS-V1.0.md#§4.1.3`](file:///e:/workspace/SRS-from-PRD/SRS-V1.0.md) — REQ-FUNC-019 (필요 인원 역산 기능)
- 연계 테이블: `DB-007` (Forecast 출고수 산출 완료 후 참조), `DB-009` (Workforce_Plan 저장)

## ✅ Task Breakdown (실행 계획)

### 1. 비즈니스 로직 산출 함수
- [ ] `app/domain/workforce/services/calculator.py` 작성:
  ```python
  import math
  from sqlalchemy.ext.asyncio import AsyncSession
  from app.adapters.persistence.forecast_repository import get_total_predicted_qty_for_tomorrow
  from app.domain.workforce.models import WorkforcePlan

  async def calculate_workforce_demand(session: AsyncSession, tenant_id: str, center_id: str, target_date: str) -> WorkforcePlan:
      # 1. 내일자 센터의 모든 예상 출고 박스 합산 조인 질의 (조회)
      total_predicted_qty: float = await get_total_predicted_qty_for_tomorrow(session, tenant_id, center_id, target_date)
      
      # 2. 테넌트 별 1인당 CAPA 환경 설정 변수 불러옴 (일단 데모 하드코딩 50박스)
      # TODO: 향후 테넌트 Setting 테이블을 뒤져서 capa 가져오도록 개선
      capacity_per_worker = 50.0 
      
      # 3. 인원 역산 (올림 처리 - 사람이 0.5명일순 없고 모자르는 것보단 나음)
      calculated_workers = math.ceil(total_predicted_qty / capacity_per_worker)
      
      # 4. 엔터티 DTO 조립 반환 (이후 레포지토리나 Worker가 받아서 DB에 Save함)
      plan = WorkforcePlan(
          tenant_id=tenant_id,
          center_id=center_id,
          target_date=target_date,
          predicted_shipments=int(total_predicted_qty),
          capacity_per_worker=capacity_per_worker,
          recommended_workers=calculated_workers
      )
      
      return plan
  ```

### 2. ETL 스케줄 Worker 연계
- [ ] 예측 추론(`F1-003`) Job 이 종료된 직후에 후속으로 돌아갈 수 있도록 `ETL-001` 오케스트레이터의 타임테이블 순서를 정비한다.

## 🧪 Acceptance Criteria (BDD/GWT)

**Scenario 1: 올림연산(Ceil) 정확도 검토**
- Given: 153개의 예측 출고건, 1인당 50개의 CAPA (몫: 3.06)
- When: `calculate_workforce_demand` 통과
- Then: 반올림하여 3명이 도출되면 작업이 미달되므로, `math.ceil` 이 작동해 반드시 4명의 `recommended_workers` 가 온전히 도출되는 안전성 확보 여부.

**Scenario 2: 방어 코드(Zero Division 보완)**
- Given: 휴점 공사 등으로 인한 `capacity_per_worker` 세팅값 = 0 
- When: 계산 시도 
- Then: `ZeroDivisionError` 시스템 파괴 에러로 가지 못하도록 서비스 내의 If 안전망(`if capa == 0: return 0`) 혹은 Pydantic Validation 망이 사전 차단한다.

## ⚙️ Technical Constraints
- 단순 Rule-based 산술 이지만, 시스템 전체에서 이 변수를 가장 핵심지표로 삼아 현업이 결제/전송(`F3-W03`)하므로 Float 유실 등 버그가 없도록 단위 테스트(`@pytest.mark.parametrize`)를 꼼꼼히 구성한다.

## 🏁 Definition of Done (DoD)
- [ ] `get_total_predicted_qty_for_tomorrow` 의 합산 조인 Repository 질의가 작성되었고 끌어다 썼는가?
- [ ] Zero Division 에러를 방어하는 예외 코드가 반영?
- [ ] 반올림(`round`)이 아닌 무조건 올림(`math.ceil`) 로직이 강제 점검되었나?
