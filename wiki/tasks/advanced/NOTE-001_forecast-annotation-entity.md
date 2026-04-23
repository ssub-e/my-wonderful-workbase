---
name: Feature Task
about: SRS 기반의 구체적인 개발 태스크 명세
title: "[Feature] NOTE-001: 예측 포인트 주석(Annotation) 엔티티 및 스레드형 메모 로직"
labels: 'feature, backend, database, priority:medium'
assignees: ''
type: task
tags: [task, advanced]
created: 2026-04-22
---
## 🎯 Summary
- 기능명: [NOTE-001] 예측 데이터 포인트별 협업용 주석(Annotation) 엔진
- 목적: REQ-FUNC-028 확장. 특정 날짜의 예측값이 평소보다 높거나 낮을 때, MD가 그 이유(예: "연예인 유튜브 노출됨")를 데이터 포인트에 직접 박아 넣고 동료와 스레드 형태로 소통할 수 있는 영속성 계층을 구축한다.

## 🔗 References (Spec & Context)
> 💡 AI Agent & Dev Note: 작업 시작 전 아래 문서를 반드시 먼저 Read/Evaluate 할 것.
- 데이터 모델: `FORECAST` 테이블 (FK 연계 필수)
- 아키텍처: 헥사고날 (domain/models.py 내 엔티티 정의)

## ✅ Task Breakdown (실행 계획)
- [ ] `FORECAST_ANNOTATION` 테이블 설계: id, forecast_id(FK), user_id(FK), content(Text), parent_id(Self-FK, 대댓글용), created_at
- [ ] SQLModel 기반 비동기 Repository(`AnnotationRepository`) 구현
- [ ] 다중 테넌트 격리 로직 적용: 특정 테넌트 유저는 자기 회사의 예측 주석만 조회 가능해야 함
- [ ] 주석 작성 시 자동 로그(`AUDIT_LOG`) 기록 인터셉터 연결
- [ ] 특정 예측 데이터 삭제 시 관련 주석의 Clean-up (또는 Soft-delete) 정책 수립

## 🧪 Acceptance Criteria (BDD/GWT)
Scenario 1: 예측 포인트에 새로운 메모 작성
- Given: `forecast_id='f123'` 인 예측 데이터가 존재함
- When: 유저가 "블랙 프라이데이 영향 반영됨" 이라는 메모를 작성함
- Then: `FORECAST_ANNOTATION` 테이블에 레코드가 생성되고, `forecast_id` 로 조회 시 해당 텍스트가 반환된다.

Scenario 2: 대댓글(Thread) 구조 검증
- Given: 기존에 작성된 주석 `id='a001'` 이 존재함
- When: 다른 유저가 해당 주석에 "확인했습니다" 라는 답글을 작성함 (parent_id='a001')
- Then: 트리 구조로 데이터가 저장되며, 조회 시 부모-자식 관계가 명확히 JSON에 포함된다.

## ⚙️ Technical & Non-Functional Constraints
- 보안: `tenant_id` 를 통한 데이터 고립 (타사 메모 노출 절대 금지)
- 성능: 한 페이지(차트) 조회 시 수백 개의 메모가 있을 수 있으므로 인덱싱(`forecast_id`) 및 레이지 로딩 고려
- 확장성: 향후 이미지 첨부 기능을 위해 `attachment_url` 컬럼 미리 확보(Optional)

## 💻 Implementation Snippet (Schema Idea)
```python
class ForecastAnnotation(SQLModel, table=True):
    __tablename__ = "forecast_annotation"
    
    id: Optional[uuid.UUID] = Field(default_factory=uuid.uuid4, primary_key=True)
    tenant_id: uuid.UUID = Field(foreign_key="tenant.id", index=True)
    forecast_id: uuid.UUID = Field(foreign_key="forecast.id", index=True)
    user_id: uuid.UUID = Field(foreign_key="user.id")
    
    content: str = Field(sa_column=Column(Text))
    parent_id: Optional[uuid.UUID] = Field(default=None, foreign_key="forecast_annotation.id")
    
    created_at: datetime = Field(default_factory=datetime.utcnow)
    updated_at: datetime = Field(default_factory=datetime.utcnow)

    # Relationship for thread
    replies: List["ForecastAnnotation"] = Relationship(
        sa_relationship_kwargs={"cascade": "all, delete-orphan", "remote_side": "ForecastAnnotation.id"}
    )
```

## 🏁 Definition of Done (DoD)
- [ ] 모든 Acceptance Criteria를 충족하는가?
- [ ] SQLModel 마이그레이션 파일이 생성되었는가?
- [ ] 테넌트 간 데이터 크로스 체크(Cross-tenant leakage) 방지 테스트를 통과했는가?

## 🚧 Dependencies & Blockers
- Depends on: `DB-007` (FORECAST 테이블)
- Blocks: `API-004`, `VIEW-021`
