---
name: Feature Task
about: SRS 기반의 구체적인 개발 태스크 명세
title: "[Feature] INFRA-004: 대량 시계열 데이터(Bulk Data) 적재 최적화 및 고속 적재 어댑터"
labels: 'feature, backend, infra, priority:medium'
assignees: ''
type: task
tags: [task, infrastructure]
created: 2026-04-22
---
## 🎯 Summary
- 기능명: [INFRA-004] 성능 임계치 돌파를 위한 대용량 데이터 전용 Bulk Loader
- 목적: REQ-NF-005 확장. 신규 고객사 온보딩 시 과거 3~5년치 주문 데이터(수백만 건 이상)를 한 번에 적재할 때, 일반적인 SQL INSERT는 극도로 느리므로 PostgreSQL의 `COPY` 명령 또는 SQLAlchemy의 `bulk_insert_mappings`를 활용하여 초당 1만 건 이상의 적재 성능을 확보한다.

## 🔗 References (Spec & Context)
> 💡 AI Agent & Dev Note: 작업 시작 전 아래 문서를 반드시 먼저 Read/Evaluate 할 것.
- 관련 태스크: `ADMIN-007` (Historical Data Sync)
- 기술 스택: `SQLModel` / `asyncpg` 기반

## ✅ Task Breakdown (실행 계획)
- [ ] `app/adapters/db_bulk_loader.py` 전용 어댑터 신설
- [ ] `asyncpg` 내장 `copy_records_to_table` 메서드를 활용한 고속 바이너리 적재 로직 구현
- [ ] 대량 적재 중 시스템 전체 락(Lock) 방지를 위한 테이블 파티셔닝(Partitioning) 또는 배치 쪼개기 전략 수립
- [ ] 트랜잭션 관리: 일부 실패 시 전체 롤백 또는 정밀 에러 로그(Skipped Rows) 반환 기능
- [ ] 적재 전 데이터 유효성 검사(Schema Validation)를 비동기적으로 병렬 처리하는 파이프라인 구축

## 🧪 Acceptance Criteria (BDD/GWT)
Scenario 1: 100만 건 데이터 고속 적재 성능 측정
- Given: 과거 3년치 주문 데이터 CSV 파일 (100만 라인)
- When: `INFRA-004` 로더를 통해 적재를 시작함
- Then: 전체 적재 시간이 2분 이내(약 8,000~10,000 rows/sec)에 완료되어야 함.

Scenario 2: 적재 중 중복 데이터 처리
- Given: 이미 DB에 있는 동일한 `platform_order_id` 가 포함된 데이터 뭉치
- When: Bulk Insert 수행
- Then: `ON CONFLICT DO NOTHING` 또는 `UPSERT` 로직이 효율적으로 작동하여 PK 중복 에러로 전체 작업이 중단되지 않아야 함.

## ⚙️ Technical & Non-Functional Constraints
- 자원 관리: 적재 중 메모리 사용량이 컨테이너 제한(예: 1GB)을 넘지 않도록 제너레이터(Generator) 패턴 사용
- 투명성: `INSERT_HISTORY` 테이블에 적재 시작/종료 시간, 성공 건수, 실패 사유를 기록
- 안정성: 적재 중 데이터베이스 인덱스 갱신 부하를 줄이기 위해 대량 작업 시 인덱스를 잠시 드롭(Drop) 후 재생성하는 전략 고려

## 💻 Implementation Snippet (asyncpg COPY Idea)
```python
import asyncpg

async def bulk_load_orders(conn: asyncpg.Connection, order_tuples: List[tuple]):
    """
    asyncpg의 copy 기능을 이용한 광속 데이터 적재
    """
    # column_names에 맞춰 튜플 순서가 정렬되어 있어야 함
    cols = ("id", "tenant_id", "shop_id", "platform_order_id", "order_date", "total_amount")
    
    result = await conn.copy_records_to_table(
        "order",
        records=order_tuples,
        columns=cols,
        timeout=300
    )
    return result # "COPY [N]" 형태의 문자열 반환
```

## 🏁 Definition of Done (DoD)
- [ ] 모든 Acceptance Criteria를 충족하는가?
- [ ] 10만 건 이상의 샘플 데이터를 통한 벤치마크 테스트 결과 보고서가 작성되었는가?
- [ ] `asyncpg` 의 저수준 API를 쓰더라도 테넌트 격리(`tenant_id` 주입)가 강제되는지 확인 완료?

## 🚧 Dependencies & Blockers
- Depends on: `INFRA-001`, `DB-003`
- Blocks: `ADMIN-007` (데이터 마이그레이션 도구)
