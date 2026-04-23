---
name: Feature Task
about: SRS 기반의 구체적인 개발 태스크 명세
title: "[DB] DB-004: PRODUCT + INVENTORY 테이블 스키마 + Repository"
labels: 'database, backend, priority:high, phase:1'
assignees: ''
type: task
tags: [task, database]
created: 2026-04-22
---
## 🎯 Summary
- 기능명: [DB-004] PRODUCT 카탈로그와 분리된 실시간 INVENTORY 테이블 세팅
- 목적: 예측 엔진은 특정 **SKU 단위**로 발주 예측을 수행해야 하므로(REQ-FUNC-004), 이 SKU 식별 기준이 되는 상품 마스터(`PRODUCT`)와 변동치인 재고 현황(`INVENTORY`) 엔터티를 DB에 정의하고 데이터 접근 계층(Repository)을 셋업한다.

## 🔗 References (Spec & Context)
- SRS 데이터 모델러: [`SRS-V1.0.md#§6.2`](file:///e:/workspace/SRS-from-PRD/SRS-V1.0.md) — PRODUCT, INVENTORY ERD 매핑
- SRS 재고 수집 업무: [`SRS-V1.0.md#§4.1.2`](file:///e:/workspace/SRS-from-PRD/SRS-V1.0.md) — REQ-FUNC-014 (연속적인 현재 재고량의 최신화 목적 확인)

## ✅ Task Breakdown (실행 계획)

### 1. SQLModel 스키마 통합 구성
- [ ] `app/domain/integration/models.py` 확장. 상품과 인벤토리를 1:1 관계로 연결:
  ```python
  class Product(SQLModel, table=True):
      __tablename__ = "products"
      
      id: uuid.UUID = Field(default_factory=uuid.uuid4, primary_key=True)
      shop_id: uuid.UUID = Field(foreign_key="shops.id", nullable=False)
      sku: str = Field(index=True, nullable=False, description="품목 바코드 혹은 식별 키")
      name: str = Field(nullable=False)
      category: Optional[str] = Field(default=None)
      
      # 1:1 관계 (Nullable)
      inventory: Optional["Inventory"] = Relationship(back_populates="product", sa_relationship_kwargs={"uselist": False})

  class Inventory(SQLModel, table=True):
      __tablename__ = "inventories"
      
      id: uuid.UUID = Field(default_factory=uuid.uuid4, primary_key=True)
      product_id: uuid.UUID = Field(foreign_key="products.id", unique=True, nullable=False) # 1:1 관계 강제 포인트
      current_qty: int = Field(default=0)
      safety_stock: int = Field(default=0)
      last_updated: datetime = Field(default_factory=datetime.utcnow, sa_column_kwargs={"onupdate": datetime.utcnow})
      
      product: Product = Relationship(back_populates="inventory")
  ```

### 2. DTO 구조
- [ ] `app/schemas/inventory_schema.py`:
  - `ProductInventoryRead`: 상품명과 남은수량을 한번에 응답하는 조인 모델 시나리오 구성 (대시보드 표시 등의 범용성).

### 3. Repository 인터페이스 작성
- [ ] `app/adapters/persistence/inventory_repository.py`:
  ```python
  # 새롭게 수집된 재고데이터(Upsert)
  async def upsert_inventory(session: AsyncSession, product_id: UUID, current_qty: int) -> Inventory:
      """만약 해당 product_id 의 인벤토리 라인이 있으면 Update, 없으면 Create 하도록 처리"""
      ...
  
  # 발주 임계치 미달 리포트
  async def get_low_stock_products(session: AsyncSession, shop_id: UUID) -> List[Product]:
      """current_qty < safety_stock 인 상품 리스트를 Join 하여 가져옴"""
      ...
  ```

### 4. 마이그레이션 적용
- [ ] `alembic` 생성: `create_product_and_inventory_tables`

## 🧪 Acceptance Criteria (BDD/GWT)

**Scenario 1: Upsert 기반의 최신량 덮어쓰기 및 onupdate 액션**
- Given: 재고수량이 10개인, 시간 T 기준의 `Inventory` 데이터를 삽입한 상태.
- When: Repository의 `upsert_inventory()` 로 신규 수량(5개) 업데이트를 날림.
- Then: 새로운 row가 생기지 않고 기존 row의 `current_qty=5` 로 업데이트 되며, `last_updated` 필드가 변경 당시 최신 시계열(PostgreSQL 트리거 혹은 SQLAlchemy onupdate)로 갱신된 것을 확인한다.

**Scenario 2: 상품과 재고의 1:1 제한 (무결성)**
- Given: `product_id="B"`를 참조하는 단 1개의 인벤토리 레코드가 있음
- When: 어긋난 수집 로직이 의도치 않게 DB에 `Insert` 쿼리로 같은 `product_id="B"`의 인벤토리를 하나 더 구겨 넣으려 함
- Then: DB의 Unique constraint 제약으로 트랜잭션이 실패하며 시스템에 불필요한 다중 재고 항목 발생을 원천 차단한다.

## ⚙️ Technical Constraints
- SQLAlchemy `onupdate` 키워드는 파이썬 객체 레벨의 트리거 기반이므로, DB단에서 직접 수행(수동 UPDATE 문) 될 때에도 갱신되길 바란다면 PostgreSQL 커스텀 트리거를 적용하는 대안도 고려할 것. 보편적으로는 ORM 단의 `sa_column_kwargs` 로 갈음함.

## 🏁 Definition of Done (DoD)
- [ ] 상기 1:1 Relationship 을 맺은 SQLModel 코드가 복제 작성 되었는가?
- [ ] `Inventory`의 `product_id` 외래키에 `Unique=True` 플래그가 세팅?
- [ ] Repository에 `upsert_inventory` 등 실제 수집 로직에 복붙해 쓸 함수 뼈대가 마련되었는가?
