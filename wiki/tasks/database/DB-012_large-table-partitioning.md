---
name: Feature Task
about: SRS 기반의 구체적인 개발 태스크 명세
title: "[Feature] DB-012: 대용량 과거 주문 실적 테이블 최적화를 위한 '테이블 파티셔닝(Partitioning)' 설계"
labels: 'feature, backend, db, priority:medium'
assignees: ''
type: task
tags: [task, database]
created: 2026-04-22
---
## 🎯 Summary
- 기능명: [DB-012] 성능 한계 극복을 위한 시간축 기반 테이블 샤딩/파티셔닝
- 목적: SaaS가 성장하며 수백 개 테넌트의 주문 데이터(ORDER_ITEM)가 수천만 건 이상 쌓였을 때, 단일 테이블에서의 조회 성능 저하(Index Bloat)를 방지한다. 결제 및 예측 실적 데이터를 '월 단위' 또는 '테넌트 단위'로 물리적으로 분리 저장하여 쿼리 성능을 상시 쾌적하게 유지한다.

## 🔗 References (Spec & Context)
> 💡 AI Agent & Dev Note: 작업 시작 전 아래 문서를 반드시 먼저 Read/Evaluate 할 것.
- DB 엔진: PostgreSQL 14+ (Declarative Partitioning)
- 타겟 테이블: `ORDER`, `ORDER_ITEM`, `FORECAST_DATA`

## ✅ Task Breakdown (실행 계획)
- [ ] **Partition Key 선정**: 데이터를 가장 자주 필터링하는 컬럼(`created_at` 또는 `tenant_id`)으로 파티션 키 확정
- [ ] **Table Refactoring**: 
  - 기존 통짜 테이블을 파티션 마스터(Parent) 테이블로 승격
  - 월 단위(`PARTITION BY RANGE`) 또는 해시 단위 자식 테이블 생성 스크립트 작성
- [ ] **Automation Logic**: 매월 말에 다음 달 파티션을 자동으로 생성하는 백그라운드 워커(`INFRA-003` 연계) 구축
- [ ] **Pruning Optimization**: 쿼리 시 불필요한 파티션을 읽지 않도록(Partition Pruning) 인덱스 및 쿼리 구문 가이드 작성
- [ ] **Data Migration**: 기존 데이터를 서비스 중단 없이 새 파티션 구조로 밀어넣는 롤링 마이그레이션 스크립트

## 🧪 Acceptance Criteria (BDD/GWT)
Scenario 1: 시간 기준 쿼리 성능 개선
- Given: 2년치(2400만 건)의 주문 데이터
- When: 최근 1개월 데이터를 조회함
- Then: DB는 전체 데이터를 뒤지지 않고 오직 해당 월의 파티션 테이블만 조회하여, 기존 대비 5배 이상의 응답 속도 향상이 나타나야 한다.

Scenario 2: 자동 파티션 생성
- Given: 매달 25일이 됨
- When: 시스템 워커가 동작함
- Then: 아직 존재하지 않는 다음 달(n+1)의 테이블(예: `orders_y2026_m05`)이 사전에 자동 생성되어 있어야 한다.

## ⚙️ Technical & Non-Functional Constraints
- 가용성: 파티션 생성 실패 시 실시간으로 주문 수집이 막힐 수 있으므로, 최소 3개월 치의 파티션을 미리 예약 생성하는 방어 로직 필수
- 정합성: 파티션 키 컬럼은 반드시 `Primary Key` 또는 `Unique Index` 에 포함되어야 하는 PostgreSQL 제약 사항 준수
- 운용성: 3년 이상 된 아주 오래된 파티션은 `INFRA-007`(아카이빙)과 연동하여 분리 후 삭제

## 💻 Implementation Snippet (PostgreSQL Partition Idea)
```sql
-- 마스터 테이블 생성
CREATE TABLE order_items (
    id UUID NOT NULL,
    tenant_id UUID NOT NULL,
    product_id UUID NOT NULL,
    qty INT NOT NULL,
    created_at TIMESTAMP WITH TIME ZONE NOT NULL,
    PRIMARY KEY (id, created_at) -- 파티션 키 포함 필수
) PARTITION BY RANGE (created_at);

-- 자식 테이블 예시 (2026년 4월분)
CREATE TABLE order_items_y2026_m04 
PARTITION OF order_items 
FOR VALUES FROM ('2026-04-01') TO ('2026-05-01');
```

## 🏁 Definition of Done (DoD)
- [ ] 모든 Acceptance Criteria를 충족하는가?
- [ ] `EXPLAIN ANALYZE` 를 통해 Partition Pruning 이 정상 작동함을 증명했는가?
- [ ] 파티션 관리 툴(`pg_partman` 등)의 도입 여부가 결정되었는가?

## 🚧 Dependencies & Blockers
- Depends on: `DB-003`, `INFRA-003`
- Blocks: 없음
