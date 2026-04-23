---
name: Feature Task
about: SRS 기반의 구체적인 개발 태스크 명세
title: "[Feature] NOTE-002: MD 전용 '이벤트/프로모션 캘린더' 관리 모듈"
labels: 'feature, backend, logic, priority:medium'
assignees: ''
type: task
tags: [task, advanced]
created: 2026-04-22
---
## 🎯 Summary
- 기능명: [NOTE-002] 실무자 수동 입력용 비즈니스 이벤트 캘린더
- 목적: AI가 자동으로 알 수 없는 '화주사 내부 프로모션', '라이브 커머스 일정', '창고 휴무일' 등을 MD가 직접 캘린더에 기입하고, 이 정보가 향후 예측 엔진(`ADV-001`)의 가중치 변수로 활용될 수 있도록 정규화된 이벤트 데이터를 관리한다.

## 🔗 References (Spec & Context)
> 💡 AI Agent & Dev Note: 작업 시작 전 아래 문서를 반드시 먼저 Read/Evaluate 할 것.
- 관련 태스크: `ADV-001` (프로모션 가중치 반영 엔진)
- SRS 문서: [`e:\workspace\SRS-from-PRD\SRS-V1.0.md#4.1.2 F2`](#) (프로모션 시그널 파트)

## ✅ Task Breakdown (실행 계획)
- [ ] `BUSINESS_EVENT` 테이블 설계: id, tenant_id, title, event_type(PROMO/HOLIDAY/LIVE/ETC), start_date, end_date, impact_level(1~5)
- [ ] 이벤트 기간 중복 체크 및 겹치는 스케줄에 대한 우선순위 처리 로직
- [ ] 과거 이벤트 이력(Historical Events) 데이터 벌크 임포트 기능
- [ ] `ImpactLevel` 에 따른 예측 보정치 가이드값(Base Multiplier) 매핑 테이블 구축
- [ ] 캘린더 전용 기간 조회 API (`GET /api/v1/events?start=...&end=...`)

## 🧪 Acceptance Criteria (BDD/GWT)
Scenario 1: 라이브 커머스 일정 등록
- Given: 11월 20일 19:00시에 네이버 쇼핑라이브가 계획됨
- When: MD가 해당 일정을 `ImpactLevel=5` 로 등록함
- Then: `BUSINESS_EVENT` 테이블에 저장되며, 11월 20일 예측 수행 시 해당 이벤트가 "강력한 상승 요인"으로 엔진에 전달될 준비가 완료됨.

Scenario 2: 이벤트 기간 중첩 시 처리
- Given: 'A 이벤트(11/1~11/10, Level 1)'와 'B 이벤트(11/5~11/15, Level 4)'가 겹침
- When: 해당 기간의 이벤트를 조회함
- Then: 두 이벤트가 모두 배열로 반환되며, 엔진은 더 높은 `impact_level` 인 B를 주 변수로 채택할 수 있도록 가중치를 정렬해서 제공함.

## ⚙️ Technical & Non-Functional Constraints
- 성능: 캘린더 뷰에서 한 달치 이벤트를 쿼리할 때 인덱스(`tenant_id`, `start_date`) 활용 필수
- 유연성: `event_type` 확장성을 위해 Enum 대신 별도 관리용 테이블/캐시 고려
- 정확성: `end_date` 가 `start_date` 보다 이전인 경우의 에러 핸들링 (400 Bad Request)

## 💻 Implementation Snippet (Service Logic Idea)
```python
async def get_active_events(db: AsyncSession, tenant_id: uuid.UUID, target_date: date) -> List[BusinessEvent]:
    """
    특정 날짜에 활성화된 모든 비즈니스 이벤트를 중첩 순서대로 가져옴
    """
    statement = select(BusinessEvent).where(
        BusinessEvent.tenant_id == tenant_id,
        BusinessEvent.start_date <= target_date,
        BusinessEvent.end_date >= target_date
    ).order_by(BusinessEvent.impact_level.desc())
    
    result = await db.execute(statement)
    return result.scalars().all()
```

## 🏁 Definition of Done (DoD)
- [ ] 모든 Acceptance Criteria를 충족하는가?
- [ ] 일자별 이벤트 조회 API에 대한 단위 테스트가 작성되었는가?
- [ ] `ImpactLevel` 에 대한 비즈니스 정의가 확정되었는가?

## 🚧 Dependencies & Blockers
- Depends on: `DB-001`
- Blocks: `ADV-001`, `VIEW-021` (캘린더 연동 뷰)
