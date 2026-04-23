---
name: Feature Task
about: SRS 기반의 구체적인 개발 태스크 명세
title: "[Compliance] SEC-002: 고객 데이터 유실 방어용 Soft-Delete(논리 삭제) 아키텍처"
labels: 'backend, db, security, phase:8, priority:medium'
assignees: ''
type: task
tags: [task, security]
created: 2026-04-22
---
## 🎯 Summary
- 기능명: [SEC-002] 무분별한 `DELETE` 쿼리를 억제하는 Soft Delete (논리삭제) 코어 적용
- 목적: B2B 관리자가 "실수로 우리 영업 카테고리/물류센터를 삭제했어요!" 라고 CS를 걸어올 때를 대비해, DB에서 진짜 날려버리는(Hard Delete) 대신 `deleted_at` 컬럼에 타임스탬프를 찍고 모든 조회망에서 무시(Ignore) 당하게 덮어버린다.

## 🔗 References (Spec & Context)
- 타겟 모듈: `BaseModel` (SQLModel 최상위 객체)

## ✅ Task Breakdown (실행 계획)

### 1. 베이스 클래스 필드 추가
- [ ] `app/core/models.py` (프로젝트 내 공용 Base Model 이 없다면 생성):
  ```python
  from sqlmodel import SQLModel, Field
  from datetime import datetime
  from typing import Optional

  class SoftDeleteModel(SQLModel):
      """
      모든 핵심 테이블 모듈이 이걸 상속(Inheritance) 받으면 딜리트 면역 컬럼이 생김.
      """
      deleted_at: Optional[datetime] = Field(default=None, index=True)
  ```

### 2. Repository 계층 Delete 오버라이딩
- [ ] `app/core/repository.py` (BaseRepository 계층):
  ```python
  from datetime import datetime

  class CRUDBase:
      # ...
      def remove(self, db: Session, *, id: int) -> ModelType:
          obj = db.get(self.model, id)
          # 하드 딜리트: db.delete(obj) <- 원본을 지움
          
          # 소프트 딜리트 오버라이드
          obj.deleted_at = datetime.utcnow()
          db.add(obj)
          db.commit()
          db.refresh(obj)
          return obj
          
      def get_active(self, db: Session, *, id: int) -> Optional[ModelType]:
          return db.query(self.model).filter(
              self.model.id == id,
              self.model.deleted_at == None # 핵심 차단막
          ).first()
  ```

## 🧪 Acceptance Criteria (BDD/GWT)

**Scenario 1: Soft Delete 작동 후 REST API 숨김 처리**
- Given: 대시보드에서 `id=5` 창고 마스터 데이터를 '삭제' 버튼을 눌러 API가 호출됨
- When: 내부 `CRUDBase.remove` 가 실행되어 `deleted_at` 이 세팅됨 
- Then: DB Row는 실제로 남아있으나, 대시보드의 '목록 불러오기' API 가 내부적으로 `filter(deleted_at == None)` 조건을 타고 돌기 때문에 사용자 화면에서는 마치 깨끗하게 지워진 것처럼 동작한다. (이후 개발자 DB 접근으로 원복 가능).

## ⚙️ Technical Constraints
- SQLAlchemy 레파지토리에 이 필터를 누락하는 신입 개발자들의 실수를 막기 위해, 가급적이면 ORM 최하단 스코프(Global Filter)에 바인딩하거나, SQLAlchemy의 Event Listener(예: `before_compile`)를 가혹하게 삽입하여 쿼리에 무조건 `deleted_at IS NULL` 을 강제 삽입시키는 고급 패턴 구현을 검토한다.

## 🏁 Definition of Done (DoD)
- [ ] 추상 모델 베이스 객체에 `deleted_at` 타임스탬프 필드 전파 확인?
- [ ] `.delete` 구문이 `.add(deleted_at=Now)` 구문으로 교체 오버라이딩 적용?
