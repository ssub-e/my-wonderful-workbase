---
name: Feature Task
about: SRS 기반의 구체적인 개발 태스크 명세
title: "[Feature] API-004: 협업용 메모 및 비즈니스 이벤트 CRUD 라우터"
labels: 'feature, backend, api, priority:medium'
assignees: ''
type: task
tags: [task, backend_api]
created: 2026-04-22
---
## 🎯 Summary
- 기능명: [API-004] 협업 포털(Notes/Events)용 통합 REST API 계층
- 목적: `NOTE-001`(주석)과 `NOTE-002`(캘린더) 기능을 프론트엔드와 연결하기 위한 핵심 API 라우터 집합을 구축한다. 모든 요청은 테넌트 격리(`tenant_id`)가 강제되며, 유저의 권한(Role)에 따라 쓰기/삭제 권한이 분기된다.

## 🔗 References (Spec & Context)
> 💡 AI Agent & Dev Note: 작업 시작 전 아래 문서를 반드시 먼저 Read/Evaluate 할 것.
- 기반 엔티티: `ForecastAnnotation`, `BusinessEvent`
- 보안 가이드: `AUTH-001` (JWT/RBAC), `DB-011` (RLI)

## ✅ Task Breakdown (실행 계획)
- [ ] `app/api/v1/endpoints/notes.py` 구현:
  - `POST /` : 특정 예측 포인트 주석 작성
  - `GET /forecast/{forecast_id}` : 특정 예측 포인트의 메모 스레드 전체 조회
  - `DELETE /{note_id}` : 본인 작성 메모 삭제 (Hard/Soft 선택)
- [ ] `app/api/v1/endpoints/events.py` 구현:
  - `POST /` : 비즈니스 이벤트(프로모션 등) 등록
  - `GET /` : 전체 캘린더 이벤트 조회 (Filter: period_start, period_end)
  - `PATCH /{event_id}` : 이벤트 속성(ImpactLevel 등) 수정
- [ ] FastAPI `Depedency Injection`을 통한 `CurrentTenant` 검증 미들웨어 적용
- [ ] Pydantic 기반의 엄격한 Input/Output DTO 매핑

## 🧪 Acceptance Criteria (BDD)
Scenario 1: 타사 메모 탈취 방도 확인
- Given: 테넌트 A 소속 유저가 테넌트 B의 `forecast_id` 로 주석 조회를 시도함
- When: `GET /api/v1/notes/forecast/{B_id}` 호출
- Then: 서버는 `404 Not Found` 또는 `403 Forbidden` 을 반환하여 데이터 격리를 보장한다.

Scenario 2: 대댓글 포함 스레드 응답 구조
- Given: 특정 예측에 부모-자식 관계의 주석이 3개 존재함
- When: 특정 예측의 노트 목록 조회 호출
- Then: 응답 JSON은 최상위 노트 리스트를 반환하며, 각 노트 내부의 `replies` 필드에 자식 노트가 중첩된 트리 구조로 포함되어야 한다.

## ⚙️ Technical & Non-Functional Constraints
- 성능: 캘린더 조회 시 범위 검색(Overlapping range) 성능 최적화 (p95 ≤ 200ms)
- 안정성: 메모 작성 시 빈 문자열 또는 2,000자 초과 입력에 대한 유효성 검사 (422 Unprocessable Entity)
- 로깅: 모든 C/U/D 행위는 `AUDIT_LOG`에 자동 기록됨

## 💻 Implementation Snippet (FastAPI Router)
```python
@router.post("/", response_model=NoteReadDto)
async def create_note(
    *,
    db: AsyncSession = Depends(get_db),
    note_in: NoteCreateDto,
    current_user: User = Depends(get_current_active_user)
):
    """
    새로운 예측 주석 작성 (테넌트 격리 포함)
    """
    new_note = ForecastAnnotation(
        **note_in.dict(),
        tenant_id=current_user.tenant_id,
        user_id=current_user.id
    )
    db.add(new_note)
    await db.commit()
    await db.refresh(new_note)
    return new_note
```

## 🏁 Definition of Done (DoD)
- [ ] 모든 Acceptance Criteria를 충족하는가?
- [ ] 메모/이벤트 CRUD 전반에 대한 통합 테스트(Integration Test)가 완료되었는가?
- [ ] 에러 메시지가 다국어(`I18N-002`) 규칙을 따르는가?

## 🚧 Dependencies & Blockers
- Depends on: `NOTE-001`, `NOTE-002`, `AUTH-001`
- Blocks: `VIEW-021` (프론트 연동)
