---
name: Feature Task
about: SRS 기반의 구체적인 개발 태스크 명세
title: "[Worker] F1-W02: 네이버 트렌드 데이터 수집 워커"
labels: 'worker, forecast, phase:1, priority:medium'
assignees: ''
type: task
tags: [task, misc]
created: 2026-04-22
---
## 🎯 Summary
- 기능명: [F1-W02] 네이버 검색 트렌드 지수 자동 수집 Worker
- 목적: 매일 07:00에 발동하여 `ADAPT-002` (데이터랩 어댑터)와 `DB-006` (트렌드 레포지토리)를 이어준다. 최근 30일간의 지정된 핵심 키워드 검색량 변동 지수를 Bulk UPSERT하여 시계열 예측 모델의 외부 변수로 지원한다.

## 🔗 References (Spec & Context)
- SRS 요구사항: [`SRS-V1.0.md#§4.1.1`](raw/assets/SRS-V1.0.md) — REQ-FUNC-002(네이버 검색 조회수 트렌드 반영)
- 연계 티켓: `ADAPT-002` (통신부), `DB-006` (저장소)

## ✅ Task Breakdown (실행 계획)

### 1. 트렌드 수집 Worker 비즈니스 로직
- [ ] `app/domain/forecast/workers/trend_worker.py` 작성:
  ```python
  import logging
  from datetime import date, timedelta
  from sqlalchemy.ext.asyncio import AsyncSession
  from app.core.retry_handler import with_etl_retry
  from app.core.database import AsyncSessionLocal
  from app.core.config import settings
  from app.adapters.external.naver_datalab import NaverDataLabAdapter
  from app.adapters.persistence.trend_repository import bulk_upsert_trends
  
  logger = logging.getLogger(__name__)

  @with_etl_retry(max_retries=2, base_delay_sec=60) # 네이버는 1분 슬립 권장
  async def run_trend_collection_worker():
      end_date = date.today()
      start_date = end_date - timedelta(days=30)
      logger.info(f"Starting trend collection worker ({start_date} ~ {end_date})")
      
      adapter = NaverDataLabAdapter(
          client_id=settings.NAVER_CLIENT_ID, 
          client_secret=settings.NAVER_CLIENT_SECRET
      )
      
      # TODO: 하드코딩된 필수 추적 키워드 (향후 테넌트 관리자에서 동적 조립)
      target_keywords = ["수분크림", "기저귀", "단백질보충제"]
      
      async with AsyncSessionLocal() as session:
          all_trends = []
          for keyword in target_keywords:
              try:
                  # DTO List를 돌려줌
                  trends_dto_list = await adapter.fetch_trend_index(
                      keyword=keyword, 
                      start_date=start_date.strftime("%Y-%m-%d"), 
                      end_date=end_date.strftime("%Y-%m-%d")
                  )
                  all_trends.extend(trends_dto_list)
              except Exception as e:
                  logger.error(f"Error fetching trend for {keyword}: {e}")
                  raise # DLQ Fallback을 위한 예외 토스
                  
          # Bulk DB 반영
          if all_trends:
              await bulk_upsert_trends(session, all_trends)
              await session.commit()
              logger.info(f"Trend data ({len(all_trends)} rows) successfully upserted.")
  ```

## 🧪 Acceptance Criteria (BDD/GWT)

**Scenario 1: 벌크 인서트에 의한 대량 적재**
- Given: 대상 키워드가 3개이고 30일치 조회가 지시됨 (총 90개 행 발생 예상)
- When: 어댑터들이 응답을 반환하고 `bulk_upsert_trends` 에 투입됨
- Then: ORM의 `executemany` 혹은 Bulk upsert 지원기능을 통해, 매 행을 인서트 쿼리날려 병목을 일으키지 않고 한 트랜잭션으로 빠르게 90줄이 `TREND_DATA` 적재된다.

**Scenario 2: 할당량 소진 시 로깅(재시도 포기 대응)**
- Given: 어댑터에서 하루 API 1,000건 콜 한도 초과(`QuotaExceeded`) 에러가 튀어나옴
- When: 워커 로직 단
- Then: 이 에러를 잡은 재시도 핸들러가 최종 실패 DLQ 슬랙 경보를 발동하여 운영진에게 네이버 API 결제/갱신을 요구하게 만든다.

## ⚙️ Technical Constraints
- 네이버 API의 일일 Limit가 크지 않을 경우, 어제와 오늘치만 가져오는 델타(Delta) 인서트 방식으로 전환할 수 있도록 워커의 날짜 파라미터 제어를 모듈화(유연성 확보)해 둔다.

## 🏁 Definition of Done (DoD)
- [ ] Start/End Date를 유연하게 조절하며 어댑터를 호출하는 코드 블럭 작성?
- [ ] Bulk 저장소 연동 코드 결합 완료?
- [ ] `@with_etl_retry` 부착 여부 확인?
