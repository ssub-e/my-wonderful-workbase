---
name: Feature Task
about: SRS 기반의 구체적인 개발 태스크 매뉴얼
title: "[DB] DB-003: ORDER + ORDER_ITEM 테이블 스키마 + SQLModel 모델 + Repository"
labels: 'database, backend, priority:high, phase:1'
assignees: ''
type: task
tags: [task, database]
created: 2026-04-22
---
## 🎯 Summary
- 기능명: [DB-003] ORDER 및 ORDER_ITEM 엔터티 스키마 및 레포지토리 구축
- 목적: 매출 데이터 분석 및 프로모션 시그널(주문량 폭증) 감지를 위해 카페24 등으로부터 수집되는 일별 주문 내역 데이터를 저장한다. 향후 1회 주문이 수 십개의 상품(`ORDER_ITEM`) 라인을 지닐 수 있으므로 정규화된 마스터-디테일 구조를 띤다.

## 🔗 References (Spec & Context)
- SRS ERD: [`SRS-V1.0.md#§6.2`](raw/assets/SRS-V1.0.md) — ORDER/ORDER_ITEM 관계
- 프로모션 시그널: [`SRS-V1.0.md#§4.1.2`](raw/assets/SRS-V1.0.md) — REQ-FUNC-017 (is_promotion 탐지 필드 연관)

## ✅ Task Breakdown (실행 계획)

### 1. SQLModel 스키마 모델링
- [ ] `app/domain/integration/models.py` (또는 `app/domain/order/models.py`) 확장:
  ```python
  from sqlmodel import Relationship
  from decimal import Decimal

  class Order(SQLModel, table=True):
      __tablename__ = "orders"
      
      id: uuid.UUID = Field(default_factory=uuid.uuid4, primary_key=True)
      shop_id: uuid.UUID = Field(foreign_key="shops.id", nullable=False)
      platform_order_id: str = Field(index=True, nullable=False, description="원천 시스템의 원래 주문 번호 (ex. 20231221-00001)")
      order_date: datetime = Field(index=True, nullable=False)
      total_amount: Decimal = Field(default=0, max_digits=12, decimal_places=2)
      is_promotion: bool = Field(default=False)
      
      # 양방향 관계 설정 (Cascaded 제어)
      items: List["OrderItem"] = Relationship(back_populates="order", sa_relationship_kwargs={"cascade": "all, delete-orphan"})

  class OrderItem(SQLModel, table=True):
      __tablename__ = "order_items"
      
      id: uuid.UUID = Field(default_factory=uuid.uuid4, primary_key=True)
      order_id: uuid.UUID = Field(foreign_key="orders.id", nullable=False)
      product_id: Optional[uuid.UUID] = Field(foreign_key="products.id", nullable=True) # 상품 스펙상 미등록 상품 방어용 빈값 허용
      quantity: int = Field(default=1)
      unit_price: Decimal = Field(default=0, max_digits=10, decimal_places=2)

      order: Order = Relationship(back_populates="items")
  ```

### 2. 고유 제약조건 (멱등성 인덱스) 확인
- [ ] `shop_id` + `platform_order_id` 쌍이 논리적으로 유일하도록 DB 레벨 인덱스를 조정할 것.

### 3. Pydantic DTO (입출력 용)
- [ ] `app/schemas/order_schema.py`:
  - `OrderItemRead` 매핑 모델.
  - `OrderReadWithItems`: 발주 예측 참조용이나 대시보드 리턴을 대비하여, Order와 Item List를 한 번에 직렬화하는 모델.

### 4. Repository (DAL 추상화)
- [ ] `app/adapters/persistence/order_repository.py`:
  ```python
  # 존재 여부를 묻는 방어 쿼리 (ETL 봇 멱등성 보장용)
  async def check_order_exists(session: AsyncSession, shop_id: UUID, platform_order_id: str) -> bool: ...
  
  # 주문 벌크 저장 구문
  async def create_order_with_items(session: AsyncSession, order_data: CreateOrderDTO, items: List[CreateOrderItemDTO]) -> Order: ...
  
  # 프로모션 감지 플래그 일괄 업데이트 (REQ-FUNC-017)
  async def update_promotion_flag(session: AsyncSession, order_id: UUID, is_promotion: bool) -> None: ...
  ```

## 🧪 Acceptance Criteria (BDD/GWT)

**Scenario 1: 모델 Cascade Delete 정상 작동**
- Given: 부모 `Order` 인스턴스 하나와 묶인 자식 `OrderItem` 3건이 DB에 영속화.
- When: Repository나 SQL로 `delete` Order 커맨드 작동.
- Then: 연관된 `OrderItem` 3건도 동시에 무결성 위반이나 고아 객체화 없이 파기된다. (cascade delete 동작 검증)

**Scenario 2: 데이터 무결성을 위한 스키마 자료형 정확도**
- Given: `total_amount = 12000.55` 라는 `order` 객체가 주어짐.
- When: DB에 저장하고 재조회함.
- Then: 부동소수점 오차 없이 정확히 `Decimal('12000.55')` 가 리턴되어 회계산출 오류를 막는다.

## ⚙️ Technical Constraints
- SQLAlchemy `Decimal` 타입은 C-API 성능 및 회계 정밀도를 위한 필수조건이다. 절대 `Float`을 쓰지 않는다.
- `OrderItem.product_id`를 느슨한 `Optional`(Nullable)로 둠으로써, 상품카탈로그(PRODUCT) 수집 스케줄과 주문 수집 스케줄이 역전되어도 시스템이 Crash 나지 않고 수용하게 설계한다.

## 🏁 Definition of Done (DoD)
- [ ] Order, OrderItem의 `Relationship` 과 Cascade 파라미터가 명시된 SQLModel 반영?
- [ ] `shop_id`, `platform_order_id` 복합키 기반의 중복 수집 필터 로직( Repository `check_order_exists`) 작성 완료?
- [ ] 금액 표기에 `Decimal` 타입 적용?
- [ ] Alembic Revision 생성 성공?
