---
name: Feature Task
about: SRS 기반의 구체적인 개발 태스크 명세
title: "[Feature] INFRA-006: 분석 및 리포트용 Read-Replica DB 연동 및 쿼리 분리"
labels: 'feature, backend, infra, priority:medium'
assignees: ''
type: task
tags: [task, infrastructure]
created: 2026-04-22
---
## 🎯 Summary
- 기능명: [INFRA-006] 마스터 DB 부하 완화를 위한 읽기 전용 복제본(Read Replica) 도입
- 목적: REQ-NF-001 보장. 수백만 건의 대량 데이터 조회나 복잡한 통계 쿼리(리포트 생성, XAI 분석 등)가 마스터 DB에서 실행될 경우, 쓰기 작업(주문 수집, 예측값 저장)의 지연을 초래할 수 있다. 이를 방지하기 위해 분석 작업 전용 읽기 전용 DB 인스턴스를 연결하고 어플리케이션 레이어에서 쿼리를 분기한다.

## 🔗 References (Spec & Context)
> 💡 AI Agent & Dev Note: 작업 시작 전 아래 문서를 반드시 먼저 Read/Evaluate 학 것.
- 인프라 제약: PostgreSQL (AWS RDS / Supabase Read Replica)
- 아카텍처: SQLAlchemy `AsyncSession` 라우팅

## ✅ Task Breakdown (실행 계획)
- [ ] `READ_REPLICA_URL` 환경 변수 신설 및 `Settings` 연동
- [ ] **Database Router** 구현: 
  - `GET` 요청 또는 `AnalyticService` 호출 시에는 Read-Replica 세션 사용
  - `POST/PUT/PATCH/DELETE` 요청 시에는 Master 세션 사용
- [ ] 리포트 생성기(`REPORT-001`) 및 대시보드 통계 API(`DASH-001`)의 DB 세션 의존성을 `read_only_db`로 교체
- [ ] 복제 지연(Replication Lag) 모니터링: 지연 시간이 5초 이상일 경우 마스터로 자동 폴백하는 안전 장치 구축

## 🧪 Acceptance Criteria (BDD/GWT)
Scenario 1: 읽기/쓰기 쿼리 자동 분기 확인
- Given: 시스템 부하가 없는 상태
- When: 새로운 주문 1건을 생성(`POST`)하고, 전체 통계(`GET`)를 조회함
- Then: 주문 생성 로그에는 Master 호스트 IP가 기록되고, 통계 조회 로그에는 Read-Replica 호스트 IP가 기록된다.

Scenario 2: 리포트 생성 중 마스터 가동성 유지
- Given: 대형 화주사가 3년치 재고 리포트 생성을 시도하여 고부하 쿼리가 실행 중임
- When: 다른 유저가 로그인하거나 정보를 수정함
- Then: 마스터 DB는 리포트 쿼리에 영향을 받지 않으므로, 유저의 수정 작업이 즉각(`p99 < 100ms`) 처리되어야 함.

## ⚙️ Technical & Non-Functional Constraints
- 정합성: 읽기 전용 DB는 마스터와 비동기 동기화되므로, 방금 수정한 값이 리포트에 즉시 안 보일 수 있음을 유저에게 명시 (Eventually Consistent)
- 고가용성: Read-Replica 장애 시 어플리케이션 중단 없이 마스터로 쿼리를 재전송하는 Failover 세션 구현
- 확장성: 향후 트래픽 증가 시 n개의 Read-Replica를 라운드 로빈(Round-robin)으로 연결 가능하도록 설계

## 💻 Implementation Snippet (SQLAlchemy Session Routing)
```python
async def get_db_session(write: bool = False):
    """
    요청 성격에 따라 마스터 또는 복제본 세션을 반환하는 DI
    """
    if write or settings.FORCE_MASTER:
        async with async_master_session() as session:
            yield session
    else:
        try:
            async with async_replica_session() as session:
                yield session
        except Exception:
            # 복제본 장애 시 마스터로 폴백
            async with async_master_session() as session:
                yield session
```

## 🏁 Definition of Done (DoD)
- [ ] 모든 Acceptance Criteria를 충족하는가?
- [ ] `REPORT-001` 태스트에 대한 DB 부하 분산 벤치마크 테스트 결과가 작성되었는가?
- [ ] 복제본 지연 시간(Replication Lag)에 대한 알람 임계치가 설정되었는가?

## 🚧 Dependencies & Blockers
- Depends on: `INFRA-001`, `REPORT-001`
- Blocks: 없음
