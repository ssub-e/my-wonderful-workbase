---
name: Feature Task
about: SRS 기반의 구체적인 개발 태스크 명세
title: "[Worker] F2-W03: 쇼핑몰 주문(ORDER) 내역 수집 자동화 워커"
labels: 'worker, integration, phase:1, priority:critical'
assignees: ''
type: task
tags: [task, misc]
created: 2026-04-22
---
## 🎯 Summary
- 기능명: [F2-W03] 카페24 쇼핑몰 주문 수집 파이프라인 Worker
- 목적: 매일 아침(또는 특정 주기마다) 테넌트 소유의 채널(Shop) 접속 정보를 끌어와, 신규 변경되거나 결제완료된 원천 발주량(`ORDER`, `ORDER_ITEM`)을 `DB-003` 에 적재한다. 이 데이터는 예측 엔진이 훈련할 가장 메인인 '사건 발생 Target 데이터' 다.

## 🔗 References (Spec & Context)
- SRS 요구사항: [`SRS-V1.0.md#§4.1.2`](raw/assets/SRS-V1.0.md) — REQ-FUNC-013 (주문 수집 자동화)
- 연관 태스크: `ADAPT-003` (카페24 연동망), `DB-002` (각종 테넌트샵 권한정보), `DB-003` (주문 보관함)

## ✅ Task Breakdown (실행 계획)

### 1. 주문 Pulling 파이프라인 구축
- [ ] `app/domain/integration/workers/order_worker.py`:
  ```python
  import logging
  from datetime import datetime, timedelta
  from sqlalchemy.ext.asyncio import AsyncSession
  from app.core.retry_handler import with_etl_retry
  # 의존성 임포트 (BaseRepository, Cafe24Adapter, OrderRepo 등)
  
  logger = logging.getLogger(__name__)

  @with_etl_retry(max_retries=3, base_delay_sec=15)
  async def run_order_collection_worker():
      """시스템에 존재하는 연결된 모든 Shop을 순회하며 어제 발생한 주문을 수집함"""
      start_datetime = datetime.now() - timedelta(days=1)
      end_datetime = datetime.now()
      
      async with AsyncSessionLocal() as session:
          # 1. 대상 샵 열람
          shops = await get_all_connected_shops(session) # status='connected'
          
          for shop in shops:
              try:
                  adapter = Cafe24Adapter(mall_id=shop.platform_mall_id, access_token=shop.oauth_token)
                  
                  # 2. 어댑터 호출 (분당 API 제한 제어 로직은 어댑터 내부에 존재함)
                  orders_dto_list = await adapter.get_orders(start_datetime.isoformat(), end_datetime.isoformat())
                  
                  # 3. 중복 수집 필터링 및 영속화 (DB-003 Repository 활용)
                  inserted_cnt = 0
                  for order_dto in orders_dto_list:
                      if not await check_order_exists(session, shop.id, order_dto.platform_order_id):
                          await create_order_with_items(session, order_dto, order_dto.items)
                          inserted_cnt += 1
                          
                  # 4. 상점 최종 동기화 시간 마킹
                  await update_last_sync(session, shop.id, datetime.utcnow())
                  logger.info(f"Shop [{shop.id}]: UPSERT {inserted_cnt} orders.")
                  
              except Exception as e:
                  logger.error(f"Shop [{shop.id}] order pull failed: {e}")
                  # Exception을 전파하여 Retry 잡동 유도 (단, 성공한 타 샵 트랜잭션을 보호할지 정책 결정 필)
                  raise 
                  
          await session.commit()
  ```

## 🧪 Acceptance Criteria (BDD/GWT)

**Scenario 1: 멱등성이 보장된 중복 주문 필터링**
- Given: 대상 샵 내에 이미 수집 완료된 `platform_order_id=ord_999` 문서기 등록됨. 
- When: 다음날 스케줄러가 안전을 위해 날짜를 넉넉하게 덮어 수집(`get_orders`)하면서 `ord_999` 전문이 다시 파싱되어 어댑터로 들어옴
- Then: 루프문 내부의 `check_order_exists()` 방어선 덕분에 불필요한 DB `Insert` 쿼리와 Primary Key 위반 충돌 에러가 생략되며 매끄럽게 넘어간다.

**Scenario 2: 카페24 토큰 만료 동적 복구 (시나리오 확정)**
- Given: 반복문 도중 특정 샵의 `oauth_token` 이 2시간이 지나 만료 에러(401)를 뱉음
- When: 어댑터 호출 또는 예외 핸들링에서 토큰 재발급(Refresh) 함수가 트리거됨
- Then: DB의 refresh_token값을 기반으로 새 엑세스 토큰이 세팅된 뒤, 실패했던 주문 수집 구간부터 재시도하여 중단없는 Worker 배치를 이룩한다.

## ⚙️ Technical Constraints
- Tenant 병렬 처리: 다수 시스템에 샵(Shop) 이 수 십개가 넘어갈 경우 `for shop in shops:` 를 직렬 수행(Sync await) 하면 밤새 배치가 돌 수 있음. 향후 `asyncio.gather` 를 활용하여 샵 단위 병렬 파이프라인 처리를 염두에 두고 코드를 짠다.

## 🏁 Definition of Done (DoD)
- [ ] 여러 테넌트 샵을 순회 조회하여 Order API를 호출하는 로직 완성?
- [ ] 이중 수집 방어 로직(`check_exists`) 및 `create_order` 매핑 처리 확인?
- [ ] 루프 종료 후 대상 샵에 대해 `last_sync` 마킹 처리가 구현되었나?
