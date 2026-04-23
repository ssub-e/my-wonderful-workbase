---
name: Feature Task
about: SRS 기반의 구체적인 개발 태스크 명세
title: "[Worker] F1-W01: 기상청 단기예보 자동 수집 워커"
labels: 'worker, forecast, phase:1, priority:high'
assignees: ''
type: task
tags: [task, misc]
created: 2026-04-22
---
## 🎯 Summary
- 기능명: [F1-W01] 단기예보 자동 수집 Worker
- 목적: 매일 06:00 스케줄에 맞춰 동작하는 단위 작업 블럭(Worker). `ADAPT-001` (기상청 어댑터)을 호출하여 내일의 날씨 예보를 획득하고, `DB-005` (WEATHER_DATA 레포지토리)에 UPSERT 하여 향후 XAI 예측 엔진 파이프라인이 즉시 가져다 쓸 수 있는 공용 Feature Data를 적재한다.

## 🔗 References (Spec & Context)
- SRS 요구사항: [`SRS-V1.0.md#§4.1.1`](raw/assets/SRS-V1.0.md) — REQ-FUNC-001(기상 데이터 수집), REQ-FUNC-003(24시간 기상 Fallback 캐시)
- 의존성 태스크: `ADAPT-001` (통신단), `DB-005` (저장단), `ETL-002` (재시도 정책)

## ✅ Task Breakdown (실행 계획)

### 1. 수집 Worker 함수 구현
- [ ] `app/domain/forecast/workers/weather_worker.py` 구성:
  ```python
  import logging
  from datetime import date, timedelta
  from sqlalchemy.ext.asyncio import AsyncSession
  from app.core.retry_handler import with_etl_retry
  from app.core.database import AsyncSessionLocal
  from app.adapters.external.weather_api import WeatherAPIAdapter
  from app.adapters.persistence.weather_repository import upsert_weather_data
  from app.adapters.persistence.cache_service import set_cache
  
  logger = logging.getLogger(__name__)

  @with_etl_retry(max_retries=3, base_delay_sec=30)
  async def run_weather_collection_worker():
      target_date = date.today() + timedelta(days=1)
      logger.info(f"Starting weather collection worker for {target_date}...")
      
      adapter = WeatherAPIAdapter(api_key=settings.WEATHER_API_KEY)
      
      # TODO: 현 MVP에서는 [서울, 부산, 용인] 3개 센터 기준으로 루프화
      regions = ["seoul", "busan", "yongin"]
      
      async with AsyncSessionLocal() as session:
          for region in regions:
              try:
                  # 1. API Fetch
                  weather_dto = await adapter.fetch_forecast(region, target_date.strftime("%Y%m%d"))
                  
                  # 2. Redis 캐시 폴백 세팅 (REQ-FUNC-003) -> 익일 조회를 위해 24h+ TTL 
                  await set_cache(f"fallback:weather:{region}:{target_date}", weather_dto.model_dump(), expire_seconds=172800)
                  
                  # 3. DB 영구 적재
                  await upsert_weather_data(session, weather_dto)
                  
              except Exception as e:
                  logger.error(f"Failed to fetch {region}: {e}")
                  raise # 재시도 데코레이터 발동을 위해 throw
                  
          await session.commit()
          logger.info("Weather collection successfully completed.")
  ```

## 🧪 Acceptance Criteria (BDD/GWT)

**Scenario 1: 정상 수집에서 DB 저장까지의 파이프라인 (E2E)**
- Given: 오전 06시 배치가 발동함 
- When: `run_weather_collection_worker()` 펑션이 구동되어 서울 등 3개 권역을 순회하며 어댑터를 On 함
- Then: 트랜잭션이 성공적으로 종결되며, 캐시에 폴백 데이터가 남고, Postgres `WEATHER_DATA` 테이블에 내일자 데이터가 3줄(UPSERT) 적재된다.

**Scenario 2: Fallback 캐싱 확인 (REQ-FUNC-003 방어)**
- Given: 위 워커 파이프라인이 도중 에러 없이 완료된 직후
- When: 분리된 쉘에서 `redis-cli get "fallback:weather:seoul:2026-05-02"` 를 조작함
- Then: 직렬화된 JSON 날씨 기온 데이터가 조회되어, 내일자 API 장애 시 메인 프로세스가 재활용할 수 있는 방호벽이 완성됨을 확인한다.

## ⚙️ Technical Constraints
- 어댑터, 세션 변수, 트랜잭션 커밋이 하나의 함수 블럭 안에서 원자적(Atomic)으로 동작하게 통제할 것.
- 루프 중 1개 지역만 실패하더라도 Exception을 던져 재시도를 유도하되, UPSERT 구문 덕택에 이미 성공한 지역은 다시 수집되어도 멱등성으로 덮어씌워지므로 무결성 충돌이 일어나지 않음을 보장한다.

## 🏁 Definition of Done (DoD)
- [ ] 기상청 Adapter 호출부터 Repository 저장까지 연결되는 파이프라인 기능 완성?
- [ ] 실패 시 동작하는 `@with_etl_retry` 부착?
- [ ] 성공 시 24h 이상 캐시에 머무르는 Fallback 적재 코드 반영?
