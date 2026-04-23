---
name: Feature Task
about: SRS 기반의 구체적인 개발 태스크 명세
title: "[API] ADMIN-006: 감사 추적 로그(Audit Log) 열람 API"
labels: 'api, admin, security, phase:4, priority:low'
assignees: ''
type: task
tags: [task, admin]
created: 2026-04-22
---
## 🎯 Summary
- 기능명: [ADMIN-006] 관리자용 활동 로그 및 보안 이력 페이징 열람 API
- 목적: B2B 엔터프라이즈 기능(REQ-NF-021 규제 준수). 회사 내 누가 상품 CAPA를 바꾸었고, 누가 수작업 동기화를 일으켰으며, 어떤 백그라운드 에러가 났는지 관리자 로그 패널에 시간순으로 노출해준다.

## 🔗 References (Spec & Context)
- 로그 적재 테이블: `DB-010` (`AUDIT_LOG` 및 JSON 속성)
- 요구 요건: REQ-NF-021 (보안 및 감사 로그 6개월 열람)

## ✅ Task Breakdown (실행 계획)

### 1. JSON 포함 형태의 DTO 작성
- [ ] `app/schemas/admin_schema.py` 내 추가:
  ```python
  from pydantic import BaseModel
  from typing import Any, Dict, Optional
  
  class AuditLogItem(BaseModel):
      id: str
      user_email: Optional[str] = None # user_id 조인 후 노출
      action: str
      entity_type: str
      changes: Optional[Dict[str, Any]] = None # Json 필드 그대로 릴레이
      created_at: str

  class AuditLogListResponse(BaseModel):
      data: List[AuditLogItem]
      total_count: int
      page: int
  ```

### 2. 페이지네이션 페치 라우터 구비
- [ ] `app/api/endpoints/admin.py`:
  ```python
  from fastapi import Query

  @router.get("/audit-logs", response_model=AuditLogListResponse)
  async def get_tenant_audit_logs(
      page: int = Query(1, ge=1),
      size: int = Query(20, ge=5, le=100),
      action_filter: Optional[str] = None, # "UPDATE", "ETL_FAIL" 등 필터링 용
      tenant_id: str = Depends(get_current_tenant),
      session: AsyncSession = Depends(get_session)
  ):
      """ 
      해당 테넌트에서 발생한 모든(또는 필터링된) 액션 로그를 최신순 정렬하여 오프셋 기반 리턴
      """
      skip = (page - 1) * size
      logs, total = await fetch_audit_logs_paginated(
          session, tenant_id, skip=skip, limit=size, action_filter=action_filter
      )
      
      mapped = [AuditLogItem(
          id=str(log.id),
          user_email=log.user_email_cache, # 조인 쿼리 혹은 비정규화된 이메일 
          action=log.action,
          entity_type=log.entity_type,
          changes=log.changes,
          created_at=log.created_at.isoformat()
      ) for log in logs]
      
      return AuditLogListResponse(data=mapped, total_count=total, page=page)
  ```

## 🧪 Acceptance Criteria (BDD/GWT)

**Scenario 1: JSON 직렬화(Serialization) 무결성**
- Given: PostgreSQL 의 JSONB 컬럼에 Dict가 아닌 이상한 텍스트나 None 값이 끼어들어간 레코드
- When: 이 API가 작동해 리스폰스를 만듦
- Then: Pydantic 의 `Dict[str, Any]` 파싱 룰에 위배되는 객체는 안전하게 None 으로 변환시키거나 빈 객체(`{}`)로 핸들링되어 HTTP 500 에러를 유발하지 않고 프론트로 전송된다.

## ⚙️ Technical Constraints
- 오프셋 기반 페이지네이션(`OFFSET skip LIMIT limit`)은 데이터가 수백만건으로 불어난 1년 뒤엔 매우 느려질 수 있다 (커서 기반 Pagination 으로의 교체 여지를 개발 인계 문서에 남길 것).

## 🏁 Definition of Done (DoD)
- [ ] 페이징 처리(`skip`, `limit`)를 동반한 라우터 작성 완료?
- [ ] JSON 필드를 오류 없이 반환하는 DTO 매핑 룰 반영 확인?
