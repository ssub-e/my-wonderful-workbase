---
name: Feature Task
about: SRS 기반의 구체적인 개발 태스크 명세
title: "[Feature] SEC-005: 무차별 대입 방지 및 Redis 기반 API 속도 제한(Rate Limiting)"
labels: 'feature, backend, security, priority:high'
assignees: ''
type: task
tags: [task, security]
created: 2026-04-22
---
## 🎯 Summary
- 기능명: [SEC-005] 시스템 가용성 보호를 위한 전역 Rate Limiter 및 Brute-force 공격 차단
- 목적: REQ-NF-012 보장. 특정 IP나 계정에서 비정상적으로 많은 요청을 보낼 때(예: 초당 1,000번 로그인 시도), Redis를 활용하여 일시적으로 차단하거나 지연 응답을 줌으로써 시스템 리소스 고갈을 막고 무작위 대입 공격으로부터 유저 계정을 보호한다.

## 🔗 References (Spec & Context)
> 💡 AI Agent & Dev Note: 작업 시작 전 아래 문서를 반드시 먼저 Read/Evaluate 할 것.
- 인프라 의존성: `INFRA-002` (Redis)
- 프레임워크: `SlowAPI` (FastAPI용) 또는 커스텀 Redis Lua Script

## ✅ Task Breakdown (실행 계획)
- [ ] Redis 기반의 `RateLimitMiddleware` 구현 (Sliding Window 알고리즘 채택)
- [ ] 정책 1: **로그인 엔드포인트** (IP당 분당 5회 제한, 초과 시 10분 차단)
- [ ] 정책 2: **일반 API 호출** (API-Key/JWT당 초당 20회 제한)
- [ ] 정책 3: **PDF 생성 등 고부하 작업** (테넌트당 분당 2회 제한)
- [ ] 차단 시 표준 응답: `429 Too Many Requests` 반환 및 `Retry-After` 헤더 주입
- [ ] 화이트리스트 기능: 본사 IP 또는 특정 VIP 테넌트는 예외 처리 가능하도록 설정

## 🧪 Acceptance Criteria (BDD/GWT)
Scenario 1: 로그인 무차별 공격 감지 및 차단
- Given: 동일한 IP에서 1분 내에 6번째 로그인 시도를 함
- When: `POST /api/v1/auth/login` 호출
- Then: 서버는 `429 Too Many Requests` 를 반환하며, 10분간 해당 IP의 모든 로그인 시도를 거부한다.

Scenario 2: 정상 유저의 일시적 폭증 대응
- Given: 합법적인 자동화 툴이 API를 초당 25회 호출함 (제한 20회)
- When: 21번째 요청부터 시도함
- Then: `Retry-After: 1` 헤더와 함께 `429` 에러가 발생하며, 유저는 1초 후 정상적으로 다시 API를 이용할 수 있다.

## ⚙️ Technical & Non-Functional Constraints
- 성능: Rate Limiting 로직 자체가 DB 부하를 주면 안 되므로 오직 Redis 메모리 내에서만 처리 (Latency ≤ 5ms)
- 신뢰성: Redis 장애 시 '제한 없음(Pass-through)'으로 작동하여 서비스 정지는 막는 Fail-safe 설계
- 모니터링: 차단된 IP 리스트 및 횟수를 `AUDIT_LOG` 에 기록하여 영업팀/보안팀에 공유

## 💻 Implementation Snippet (FastAPI Middleware Idea)
```python
import time

async def rate_limit_check(key: str, limit: int, window: int, redis_conn):
    """
    Redis Sliding Window Rate Limiter
    """
    now = time.time()
    pipe = redis_conn.pipeline()
    pipe.zremrangebyscore(key, 0, now - window)
    pipe.zadd(key, {now: now})
    pipe.zcard(key)
    pipe.expire(key, window)
    results = await pipe.execute()
    
    current_count = results[2]
    return current_count <= limit
```

## 🏁 Definition of Done (DoD)
- [ ] 모든 Acceptance Criteria를 충족하는가?
- [ ] `429` 응답 시 클라이언트 UX가 적절히 대응하는가 (경고 노출 등)?
- [ ] 분산 서버 환경(Multi-instance)에서도 Redis를 통해 전역 카운팅이 정확히 이루어지는가?

## 🚧 Dependencies & Blockers
- Depends on: `INFRA-002`
- Blocks: 없음
