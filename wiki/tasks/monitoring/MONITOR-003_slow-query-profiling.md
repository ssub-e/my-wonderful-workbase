---
name: Feature Task
about: SRS 기반의 구체적인 개발 태스크 명세
title: "[Feature] MONITOR-003: 시스템 성능 프로파일링 및 슬로우 쿼리(Slow Query) 탐지 경보"
labels: 'feature, backend, monitoring, priority:medium'
assignees: ''
type: task
tags: [task, monitoring]
created: 2026-04-22
---
## 🎯 Summary
- 기능명: [MONITOR-003] 운영 안정성을 위한 실시간 성능 병목 지점 탐지
- 목적: B2B 고객사 데이터량이 늘어남에 따라 특정 분석 API가 느려지거나 DB CPU가 튀는 현상을 사전에 방지한다. 실행 시간이 3초 이상 걸리는 쿼리나 500ms 이상 소요되는 로직을 자동으로 캡처하여 개발팀 슬랙에 공유함으로써 상시 성능 최적화 체계를 구축한다.

## 🔗 References (Spec & Context)
> 💡 AI Agent & Dev Note: 작업 시작 전 아래 문서를 반드시 먼저 Read/Evaluate 할 것.
- 인프라: PostgreSQL (`pg_stat_statements`)
- 모니터링: `MONITOR-001` (Slack Webhook), Prometheus/Grafana 연동 기초

## ✅ Task Breakdown (실행 계획)
- [ ] **SQLAlchemy Query Profiler** 미들웨어 구현:
  - 모든 DB 세션의 실행 시간을 측정하는 이벤트 리스너 연동
  - 임계값(Threshold, 예: 2.0s) 초과 시 해당 SQL문과 실행 계획(EXPLAIN)을 로그에 추출
- [ ] **FastAPI Request Timing**: 각 Endpoint 별 소요 시간을 `Server-Timing` 헤더에 주입하여 프론트엔드에서도 측정 가능하게 함
- [ ] **Slack Alert Expansion**: `MONITOR-001`을 확장하여 에러뿐 아니라 'WARNING: SLOW_QUERY' 타입의 경보를 포맷팅하여 송출
- [ ] **Resource Monitoring**: Docker 컨테이너의 메모리/CPU 사용률이 80% 초과 시 경고 발생시키는 사이드카 워커 개발 (Prometheus Exporter 활용)
- [ ] **Slow API Report**: 주간 단위로 가장 느린 상위 10개 API 리스트를 자동 집계하여 보고

## 🧪 Acceptance Criteria (BDD/GWT)
Scenario 1: 슬로우 쿼리 발생 시 실시간 알람
- Given: 100만 건 재고 테이블에 인덱스 없이 조인하는 쿼리 실행
- When: 해당 쿼리가 5초 이상 실행됨
- Then: 슬랙 채널에 "[PERF ALERT] Tenant: A, Time: 5.2s, Query: SELECT * FROM inventory..." 알람이 즉시 전송된다.

Scenario 2: 성능 지표 프론트엔드 전달
- Given: 대시보드 KPI 조회 API 호출
- When: 백엔드 로직이 200ms 소요됨
- Then: 응답 헤더에 `Server-Timing: total;dur=200, db;dur=50` 이 포함되어 브라우저 개발자 도구에서 병목 구간을 확인할 수 있다.

## ⚙️ Technical & Non-Functional Constraints
- 성능: 모니터링 로직 자체가 시스템 부하를 주지 않도록 오버헤드 최소화 (측정 지연 ≤ 2ms)
- 보안: 슬랙 전송 시 쿼리 파라미터 내의 민감 정보(유저 비번 등)는 마스킹 처리
- 정확성: 네트워크 대기 시간과 순수 DB 처리 시간을 분리하여 측정

## 💻 Implementation Snippet (Query Time Listener)
```python
@event.listens_for(Engine, "before_cursor_execute")
def before_cursor_execute(conn, cursor, statement, parameters, context, executemany):
    context._query_start_time = time.time()

@event.listens_for(Engine, "after_cursor_execute")
def after_cursor_execute(conn, cursor, statement, parameters, context, executemany):
    total_time = time.time() - context._query_start_time
    if total_time > settings.SLOW_QUERY_THRESHOLD:
        trigger_slack_perf_alert(statement, parameters, total_time)
```

## 🏁 Definition of Done (DoD)
- [ ] 모든 Acceptance Criteria를 충족하는가?
- [ ] 슬로우 쿼리 알람이 중복(Flood)되지 않도록 5분간 동일 쿼리는 1회만 알람 오게 제어되는가?
- [ ] 슬랙 메시지에 실행 계획(Explain) 링크 또는 텍스트가 포함되는가?

## 🚧 Dependencies & Blockers
- Depends on: `MONITOR-001`, `INFRA-001`
- Blocks: 없음
