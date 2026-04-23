---
name: Feature Task
about: SRS 기반의 구체적인 개발 태스크 명세
title: "[Worker] F2-W04: 상품/재고(INVENTORY) 내역 수집 자동화 워커"
labels: 'worker, integration, phase:1, priority:high'
assignees: ''
type: task
tags: [task, misc]
created: 2026-04-22
---
## 🎯 Summary
- 기능명: [F2-W04] 카페24 쇼핑몰 상품 카탈로그 및 재고량 수집 파이프라인 Worker
- 목적: 매일 아침 혹은 실시간 재배치 타이밍에, 주문과 함께 쇼핑몰에 남아있는 실 잔여 수량 데이터(INVENTORY)와 상품 스펙(PRODUCT) 메타데이터를 수집 및 동기화한다. 이는 일종의 Master DB 최신화 과정이다.

## 🔗 References (Spec & Context)
- SRS 요구사항: [`SRS-V1.0.md#§4.1.2`](raw/assets/SRS-V1.0.md) — REQ-FUNC-014 (재고 데이터 수집)
- 연관 태스크: `ADAPT-003` (카페24 연동망), `DB-004` (상품+재고 보관함)

## ✅ Task Breakdown (실행 계획)

### 1. 재고/상품 Pulling 파이프라인 구축
- [ ] `app/domain/integration/workers/inventory_worker.py`:
  ```python
  import logging
  from sqlalchemy.ext.asyncio import AsyncSession
  from app.core.retry_handler import with_etl_retry
  
  logger = logging.getLogger(__name__)

  @with_etl_retry(max_retries=3, base_delay_sec=15)
  async def run_inventory_collection_worker():
      """주문과 동일하게 시스템 내 활성 Shop을 순회하며 카탈로그와 재고를 Upsert함"""
      async with AsyncSessionLocal() as session:
          shops = await get_all_connected_shops(session)
          
          for shop in shops:
              try:
                  adapter = Cafe24Adapter(mall_id=shop.platform_mall_id, access_token=shop.oauth_token)
                  
                  # 1. 상품 리스트 Fetch
                  catalog_dto_list = await adapter.get_products()
                  
                  # 2. 카탈로그 동기화 (Upsert: 새 상품이면 Insert, 기존 스펙변화면 Update)
                  for product_dto in catalog_dto_list:
                      product_id = await upsert_product_catalog(session, shop.id, product_dto)
                      
                      # 3. 재고량 동기화 (Upsert) - DB-004 에서 생성한 Repository Func 사용
                      await upsert_inventory(
                          session=session, 
                          product_id=product_id, 
                          current_qty=product_dto.inventory_qty,
                          safety_stock=product_dto.safety_stock # (없을 경우 0 기본세팅)
                      )
                  logger.info(f"Shop [{shop.id}]: Sync {len(catalog_dto_list)} product inventory.")
                  
              except Exception as e:
                  logger.error(f"Shop [{shop.id}] inventory pull failed: {e}")
                  raise
                  
          await session.commit()
  ```

## 🧪 Acceptance Criteria (BDD/GWT)

**Scenario 1: 새 상품 등록(Insert)과 기존 상품 갱신(Update) 판별**
- Given: DB에 `SKU-100(재고30)`이 저장되어 있음. 스케줄러 파싱 데이터로 `[SKU-100(재고20), SKU-200(재고50)]` 값이 읽혀 들어옴.
- When: 내부 `upsert_product_catalog` 와 `upsert_inventory` 로직이 회전함
- Then: `SKU-100` 의 식별자 충돌(UniqueKey)로 인해 Insert 대신 Update 연산이 작용하여 재고만 20으로 갱신(last_updated 시각 교체)되며, 신규 상품 `SKU-200` 은 새로운 DB 레코드로 정상 배분/삽입되어야 한다 (Upsert 무결성 확인).

**Scenario 2: 카페24 API 페이지네이션 대응**
- Given: 한 샵(Shop)당 등록된 취급 상품의 수가 3,000개가 넘는 화주사.
- When: 어댑터의 `get_products` 호출시 한번의 Request로 전부 가져오지 못하는 룰 발생 
- Then: 어댑터 및 워커 계층에서는 next_page 커서 룰을 순회(While 루프)하면서, OOM(Out of Memory) 회피를 위해 중간중간 500개 단위로 Session.commit() 을 때려 메모리를 비워가며 적재한다.

## ⚙️ Technical Constraints
- 외래키 종속: `product_id`를 알아야 `INVENTORY`를 Upsert 할 수 있으므로, 카탈로그 동기화 로직은 필연적으로 DB에 `Product` 가 먼저 적재된 후 반환된 UUID 를 확보해서 굴러가는 직렬 구조를 따른다.

## 🏁 Definition of Done (DoD)
- [ ] 카탈로그(Product)와 재고(Inventory)의 선후행 Upsert 코드 세트 확인?
- [ ] 루프 중 발생하는 대용량 상품 페이징 처리 한계 상황 관리 고려?
- [ ] `@with_etl_retry` 부착 여부 확인?
