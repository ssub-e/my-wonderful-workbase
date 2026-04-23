---
name: Feature Task
about: SRS 기반의 구체적인 개발 태스크 명세
title: "[API] DASH-004: 물류 인력 역산치 현황 패널 API"
labels: 'api, dashboard, phase:3, priority:medium'
assignees: ''
type: task
tags: [task, frontend]
created: 2026-04-22
---
## 🎯 Summary
- 기능명: [DASH-004] 일일 적정 작업자 현황(Workforce) 반환 API
- 목적: REQ-FUNC-019, 020 (물류 현장 투입 인원 대시보드). 특정 템플릿(센터/창고 단위별)으로, F3-001 모듈을 통해 계산되어 `DB-009` 로부터 쌓인 '내일자 필요 권장 인원 수' 및 '알림톡 발송 여부 상태'를 프론트로 건네주는 단순/명료한 데이터 파이프라인.

## 🔗 References (Spec & Context)
- 인력 산출 모듈: `F3-001` (Workforce Calculator)
- 참조 DB: `DB-009` (WORKFORCE_PLAN 테이블)

## ✅ Task Breakdown (실행 계획)

### 1. 인력 패널 DTO
- [ ] `app/schemas/dashboard_schema.py`:
  ```python
  class WorkforceCenterStatus(BaseModel):
      center_id: str
      target_date: str
      predicted_total_shipments: int
      recommended_workers: int
      notification_sent: bool # notified_at 값이 null 이면 False
      notified_at_str: Optional[str] = None
  
  class WorkforcePanelResponse(BaseModel):
      data: List[WorkforceCenterStatus]
  ```

### 2. API 라우터 단
- [ ] `app/api/endpoints/dashboard.py`
  ```python
  @router.get("/workforce", response_model=WorkforcePanelResponse)
  async def get_workforce_panel(
      target_date: str = Query(..., description="YYYY-MM-DD (보통 내일 일자)"),
      tenant_id: str = Depends(get_current_tenant),
      session: AsyncSession = Depends(get_session)
  ):
      # 센터가 여러개일 수 있으므로 target_date 기준 해당 화주사의 모든 센터 Array 리턴
      plans = await get_workforce_plans(session, tenant_id, target_date)
      
      mapped = []
      for p in plans:
          mapped.append(WorkforceCenterStatus(
              center_id=p.center_id,
              target_date=str(p.target_date),
              predicted_total_shipments=p.predicted_shipments,
              recommended_workers=p.recommended_workers,
              notification_sent=bool(p.notified_at),
              notified_at_str=p.notified_at.isoformat() if p.notified_at else None
          ))
      return WorkforcePanelResponse(data=mapped)
  ```

## 🧪 Acceptance Criteria (BDD/GWT)

**Scenario 1: 알림톡 발송 상태(Boolean) 변환 검증**
- Given: DB상 `notified_at` 컬럼에 Timestamp 객체가 적혀있는 데이터 1건, NULL 인 데이터 1건
- When: 위 라우터가 매핑 처리를 수행
- Then: 프론트엔드가 쓰기 편하도록 Timestamp 기반 값이 `notification_sent: true / false` 라는 명시적인 Boolean 플래그 필드로 DTO 변환되어 서빙되어야 한다 (UI 쪽 토글 스위치 등을 위함).

**Scenario 2: 멀티 센터 그룹핑(List) 반환**
- Given: "CJ대한통운 용인"과 "롯데택배 안산" 두 개의 창고 아이디(`center_id`)를 보유한 테넌트
- When: 특정 일자에 대해 조회 API 콜
- Then: 누락없이 List 내부 Array 객체 2개가 깔끔하게 떨어져 UI 테이블 구성 조건에 충족된다.

## ⚙️ Technical Constraints
- 권한 격리: 여타 API와 동일하게 `BaseTenantRepository` 상속 객환을 활용해 오직 자신(Tenant) 소유의 창고 센터 역산치만 반환함을 1순위로 지킨다.

## 🏁 Definition of Done (DoD)
- [ ] Array(List) 형태를 반환하는 `WorkforcePanelResponse` 구조 생성?
- [ ] Timestamp 유무를 뜯어 Bool 필드로 매핑하는 Helper 로직 완성 유무?
