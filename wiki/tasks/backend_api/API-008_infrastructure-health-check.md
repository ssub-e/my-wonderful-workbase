---
name: Feature Task
about: SRS 기반의 구체적인 개발 태스크 명세
title: "[Feature] API-008: 인프라 가용성 확인을 위한 고도화된 '시스템 헬스 체크(Health Check)' API"
labels: 'feature, backend, infra, priority:low'
assignees: ''
type: task
tags: [task, backend_api]
created: 2026-04-22
---
## 🎯 Summary
- 기능명: [API-008] 인프라 모니터링 및 자동 복구를 위한 Health Check 엔드포인트
- 목적: REQ-NF-001 준수. 단순한 "200 OK"를 넘어, 시스템이 의존하는 DB, Redis, 외부 수집 API, AWS S3 등의 연결 상태를 실시간으로 스캔하여 반환한다. 이는 로드 밸런서(ALB)나 쿠버네티스(Liveness/Readiness Probe)가 장애 인스턴스를 자동으로 분리하고 복구하는 기준이 된다.

## 🔗 References (Spec & Context)
> 💡 AI Agent & Dev Note: 작업 시작 전 아래 문서를 반드시 먼저 Read/Evaluate 할 것.
- 인프라: `INFRA-001` (DB), `INFRA-002` (Redis)
- 모니터링: `MONITOR-001` (Slack Alert)

## ✅ Task Breakdown (실행 계획)
- [ ] **Standard Endpoint Implementation**: `GET /health` 라우터 구현
- [ ] **Dependency Probe Logic**:
  - `DB Check`: `SELECT 1` 쿼리 실행 속도 측정
  - `Redis Check`: `PING/PONG` 응답 대기 시간 측정
  - `S3 Check`: 테스트 객체의 `HeadObject` 메타데이터 조회 시도
  - `Worker Check`: APScheduler의 스케줄러가 정상 루프를 돌고 있는지 확인
- [ ] **Detailed JSON Response**: 전체 요약 상태(UP/DOWN)와 함께 각 구성 요소의 응답 지연 시간(Latency) 반환
- [ ] **Private Visibility**: 비즈니스 정보가 포함된 상세 상태는 특정 `X-Health-Key` 헤더를 포함한 요청에만 노출(보안)
- [ ] **Auto-Alert Integration**: 특정 컴포넌트 'DOWN' 감지 시 즉시 `MONITOR-001` 을 통해 운영팀에 긴급 알림

## 🧪 Acceptance Criteria (BDD/GWT)
Scenario 1: 정상 상태 확인
- Given: 모든 인프라가 정상 구동 중
- When: `/health` 호출
- Then: `200 OK` 와 함께 `{"status": "healthy", "checks": {"database": "up", "redis": "up", ...}}` 가 반환된다.

Scenario 2: 특정 컴포넌트(Redis) 장애 발생
- Given: Redis 서버 커넥션 끊김
- When: `/health` 호출
- Then: `503 Service Unavailable` 을 반환하여 로드 밸런서가 해당 인스턴스에 트래픽을 보내지 않도록 유도한다.

## ⚙️ Technical & Non-Functional Constraints
- 보안: 일반 유저가 상세 장애 정보를 보지 못하도록 `/health` 는 Public, `/health/details` 는 Private(Auth)으로 분리
- 성능: 헬스 체크 로직이 너무 무거우면 그 자체가 부하가 되므로, 각 Probe 마다 `Timeout: 2s` 강제 적용 및 비동기 병렬 실행
- 신뢰성: "일시적인 지연"과 "완전한 장애"를 구분하기 위해 최근 3회 실패 시에만 `DOWN` 으로 판정하는 서킷 브레이커 전략 지원

## 💻 Implementation Snippet (Async Parallel Health Check)
```python
@router.get("/health")
async def health_check(db: AsyncSession = Depends(get_db)):
    results = await asyncio.gather(
        check_db_health(db),
        check_redis_health(),
        check_s3_health(),
        return_exceptions=True
    )
    
    is_all_up = all(r is True for r in results)
    status_code = 200 if is_all_up else 503
    
    return JSONResponse(
        status_code=status_code,
        content={"status": "healthy" if is_all_up else "degraded", "timestamp": datetime.now()}
    )
```

## 🏁 Definition of Done (DoD)
- [ ] 모든 Acceptance Criteria를 충족하는가?
- [ ] AWS/Cloud 인프라의 Health Check 설정에 이 URL이 등록되었는가?
- [ ] 장애 상황 발생 시 슬랙 알림이 1분 이내에 정확히 전송되는가?

## 🚧 Dependencies & Blockers
- Depends on: `INFRA-001`, `INFRA-002`
- Blocks: `DEPLOY-001` (CD 환경 구성)
