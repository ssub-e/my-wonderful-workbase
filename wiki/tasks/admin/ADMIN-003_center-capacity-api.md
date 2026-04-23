---
name: Feature Task
about: SRS 기반의 구체적인 개발 태스크 명세
title: "[API] ADMIN-003: 각 물류센터별 적정 CAPA 수치 설정 API"
labels: 'api, admin, phase:4, priority:medium'
assignees: ''
type: task
tags: [task, admin]
created: 2026-04-22
---
## 🎯 Summary
- 기능명: [ADMIN-003] 권역 센터별 `capacity_per_worker` 산출 기준 설정 라우터
- 목적: 하드코딩 되어있던 '작업자 1인당 하루 처리량(CAPA)' 수치를 테넌트 관리자가 자신의 물류센터 숙련도 스펙에 맞추어 자유롭게 변경할 수 있도록 하는 설정 인터페이스를 구축한다.

## 🔗 References (Spec & Context)
- 역산 모듈: `F3-001` (이 수치를 기반으로 N명을 도출함)
- 기반 테이블: 개별 센터 속성을 관리하는 테넌트 분기 테이블(`TENANT_CENTER` 등) 필요시 확장. (본 문서에서는 단순화를 위해 Tenant 메타로 편입)

## ✅ Task Breakdown (실행 계획)

### 1. DTO 설계 및 테넌트 메타 확장
- [ ] `app/schemas/admin_schema.py`
  ```python
  class CenterConfigUpdate(BaseModel):
      center_name: str
      capacity_per_worker: int # ex: 50 -> 1인당 하루 50박스 처리 가능
  ```

### 2. 환경 변수 적용 라우터
- [ ] `app/api/endpoints/admin.py`
  ```python
  @router.patch("/centers/capacity")
  async def update_center_capacity(
      payload: CenterConfigUpdate,
      tenant_id: str = Depends(get_current_tenant),
      session: AsyncSession = Depends(get_session)
  ):
      """ 
      테넌트의 창고별 CAPA 수정. 
      이 값이 수정되면 다음날 아침 돌아가는 F3-001 역산 엔진은 바뀐 
      이 수치를 조견표로 활용하여 인원수를 역산하게 된다.
      """
      # update_tenant_or_center_meta()
      await create_audit_log(
          session, tenant_id, "UPDATE", "CENTER_CAPACITY", 
          f"Changed to {payload.capacity_per_worker} box/man"
      )
      return {"message": "성공적으로 저장되었습니다."}
  ```

## 🧪 Acceptance Criteria (BDD/GWT)

**Scenario 1: CAPA 변경에 따른 역산치 변동 효과**
- Given: 기존에 `capacity=50` 이어서 내일 예측 1500건에 대해 '30명'을 요구하던 시스템
- When: 어드민이 위 API를 통해 `capacity_per_worker=100` (기계 도입 등) 으로 상향 조절함
- Then: 오후 16시 스케줄러가 돌아갈 때 `F3-001` 모듈은 실시간으로 이 100을 Fetch하여 1500 / 100 = '15명' 으로 역산 결과를 줄여서 알림톡을 발송하게 된다.

**Scenario 2: 방어 검증**
- Given: 실수로 CAPA 값에 `0`이나 음수를 전달
- When: PATCH 호출 
- Then: Pydantic Validation( `Field(gt=0)` ) 에 의해 즉시 422 에러로 튕겨내어 ZeroDivisionError 백엔드 파괴를 사전 방어한다.

## ⚙️ Technical Constraints
- MVP 버전에 멀티 센터 테이블 구성이 배제되어 있다면, Tenant 테이블의 `default_capacity_per_worker` 칼럼을 추가해 단일 설정값으로 굴러가도록 간소화(Simplicity) 할 수 있다.

## 🏁 Definition of Done (DoD)
- [ ] `CenterConfigUpdate` 벨리데이션(음수, 0 제어) 포함 스크립트 작성?
- [ ] 업데이트 PATCH 엔드포인트 마련 및 Audit 로그 연계?
