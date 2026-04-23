---
name: Feature Task
about: SRS 기반의 구체적인 개발 태스크 명세
title: "[Infra] INFRA-002: Redis 캐시 레이어 및 폴백 인터페이스 연동"
labels: 'infra, backend, performance, priority:high, phase:1'
assignees: ''
type: task
tags: [task, infrastructure]
created: 2026-04-22
---
## 🎯 Summary
- 기능명: [INFRA-002] Redis (Upstash) 기반의 비동기 캐시 레이어 구성
- 목적: 기상청과 네이버 DataLab 등 외부 API 장애 발생시 제공할 임시 백업 데이터 레이어 보존(REQ-NF-010). 그리고 카페24 API 호출 제한(분당 제한)을 방어하기 위한 임시 적재용 큐(REQ-FUNC-015) 용도의 Redis 환경을 구축한다 (C-TEC-004 준수).

## 🔗 References (Spec & Context)
- SRS 제약: [`SRS-V1.0.md#§1.5.3`](file:///e:/workspace/SRS-from-PRD/SRS-V1.0.md) — C-TEC-004 Redis (Upstash 호환)
- SRS 기상 폴백: [`SRS-V1.0.md#§4.1.1`](file:///e:/workspace/SRS-from-PRD/SRS-V1.0.md) — REQ-FUNC-003, REQ-NF-010 (기상청 API 24h 캐시 폴백)
- SRS API 방어: [`SRS-V1.0.md#§4.1.2`](file:///e:/workspace/SRS-from-PRD/SRS-V1.0.md) — REQ-FUNC-015 (Rate limit 극복 캐시)

## ✅ Task Breakdown (실행 계획)

### 1. Redis 비동기 클라이언트 연결 (redis.py)
- [ ] `app/core/redis.py` 작성 (FastAPI Lifespan 통합을 위함):
  ```python
  import redis.asyncio as aioredis
  from app.core.config import settings

  redis_client = aioredis.from_url(
      settings.REDIS_URL, 
      encoding="utf-8", 
      decode_responses=True,
      socket_timeout=2.0  # 연결 장애 시 무한루프 블록킹 방어
  )
  ```

### 2. 캐시 오퍼레이션 추상화 레이어 적용
- [ ] `app/adapters/persistence/cache_service.py` 구현
  ```python
  import json
  from typing import Any, Optional
  
  async def set_cache(key: str, value: Any, expire_seconds: int = 3600) -> None:
      """dict 나 list 등 복합자료를 JSON 문자열로 직렬화하여 저장"""
      str_val = json.dumps(value)
      await redis_client.set(key, str_val, ex=expire_seconds)

  async def get_cache(key: str) -> Optional[Any]:
      """캐시 미스 또는 타임아웃 시 안전하게 None 반환(로직 지속 보장)"""
      try:
          val = await redis_client.get(key)
          return json.loads(val) if val else None
      except Exception as e:
          # 여기서 로깅만 남기고 None 반환 (Fail-safe)
          return None
  ```

### 3. FastAPI 인스턴스에 탑재
- [ ] `main.py` 의 헬스 체크 엔드포인트 `/health` 가 `redis_client.ping()` 정상 여부 체크 후 결과를 넘기게끔 확장.

## 🧪 Acceptance Criteria (BDD/GWT)

**Scenario 1: 기상 캐시 폴백 24시간 저장 시뮬레이션**
- Given: 외부 수집기에서 서울의 날씨 데이터 딕셔너리(`{"temp": 12.3}`)를 얻었다.
- When: `set_cache("weather:seoul", data, expire_seconds=86400)` 을 호출한다.
- Then: Redis에 문자열 구문이 저장되고, Redis-CLI 등에서 `TTL weather:seoul` 확인 시 86400초 카운트다운이 적용된 것이 확인된다.

**Scenario 2: Fail-safe 동작 (가용성 보장)**
- Given: 의도적으로 잘못된 `REDIS_URL`을 세팅하거나 Redis 컨테이너를 강제 종료시킨다.
- When: 어플리케이션이 `get_cache("some_key")` 를 조회를 시도한다.
- Then: `Exception`을 던져 앱 통신 장애(500 에러)를 만들지 않고, 조용히 `None` 을 반환하여 DB에서 직접 데이터를 읽어오도록 로직 흐름이 넘어간다.

**Scenario 3: 직렬화/역직렬화**
- Given: 리스트 딕셔너리 `[{"id": 1}, {"id": 2}]` 구조
- When: 캐시에 세팅 후 다시 Get을 수행함
- Then: 문자열 조작 없이 고스란히 파이썬 `List[Dict]` 타입으로 복구되어 반환된다.

## ⚙️ Technical Constraints
- 라이브러리 제약: `redis` 패키지의 `redis.asyncio` 사용. Upstash 의 경우 URL에 `rediss://` (SSL 지원) 암호화 연결 꼴을 갖는다.
- 보안: `REDIS_URL` 안의 패스워드는 하드코딩 금지.

## 🏁 Definition of Done (DoD)
- [ ] `core/redis.py` 와 `cache_service.py` 구현 및 연동 완료?
- [ ] 만료시간(TTL)과 직렬화 체계 적용?
- [ ] Exception 발생 시 `None` 반환하는 Fail-safe 구조 확보?
