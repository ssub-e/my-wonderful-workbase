---
name: Feature Task
about: SRS 기반의 구체적인 개발 태스크 명세
title: "[DB] DB-007: FORECAST + FORECAST_FACTOR 예측 및 XAI 해설 테이블 모델링"
labels: 'database, backend, priority:critical, phase:1'
assignees: ''
type: task
tags: [task, database]
created: 2026-04-22
---
## 🎯 Summary
- 기능명: [DB-007] 예측 결과(FORECAST) 및 XAI 기여도 해설(FORECAST_FACTOR) 모델링
- 목적: 본 SaaS의 엔진 코어 모델(LightGBM)로 산출된 최종 SKU별 예측 출고량과 모델 신뢰도를 보관. 또한, 해당 예측을 이끌어 낸 주요 SHAP 원인 변수(기온 저하, 전일 조회수 폭발 등)와 LLM의 자연어 해설 문장을 `1:N` 관계로 영구 저장한다 (XAI 근본 레이어).

## 🔗 References (Spec & Context)
- SRS 예측 엔진 제약: [`SRS-V1.0.md#§4.1.1`](raw/assets/SRS-V1.0.md) — REQ-FUNC-004(예측값, 신뢰도 저장), REQ-FUNC-005(SHAP 기여도 저장), REQ-FUNC-006(LLM 자연어 해설 저장)
- ERD 명세: [`SRS-V1.0.md#§6.2`](raw/assets/SRS-V1.0.md) (FORECAST 1:N FACTOR)

## ✅ Task Breakdown (실행 계획)

### 1. SQLModel 스키마 및 연관 관계 작성
- [ ] `app/domain/forecast/models.py` (핵심 대상):
  ```python
  class Forecast(SQLModel, table=True):
      __tablename__ = "forecasts"
      
      id: uuid.UUID = Field(default_factory=uuid.uuid4, primary_key=True)
      tenant_id: uuid.UUID = Field(foreign_key="tenants.id", index=True, nullable=False) # 고립성 확보
      product_id: uuid.UUID = Field(foreign_key="products.id", index=True, nullable=False) # 예측 대상 SKU
      target_date: date = Field(index=True, nullable=False) # 미래 예측일
      predicted_qty: float = Field(default=0.0) # 소수점 포함 예측량
      confidence_level: float = Field(default=0.0, description="0.0 ~ 1.0 (100%) 비율, REQ-FUNC-004")
      model_version: str = Field(max_length=50, default="lightgbm_v1")
      created_at: datetime = Field(default_factory=datetime.utcnow)
      
      # 1:N 관계
      factors: List["ForecastFactor"] = Relationship(
          back_populates="forecast", 
          sa_relationship_kwargs={"cascade": "all, delete-orphan"}
      )

  class ForecastFactor(SQLModel, table=True):
      __tablename__ = "forecast_factors"
      
      id: uuid.UUID = Field(default_factory=uuid.uuid4, primary_key=True)
      forecast_id: uuid.UUID = Field(foreign_key="forecasts.id", nullable=False)
      factor_type: str = Field(max_length=100, description="ex: weather_temp, order_trend, weekend_effect")
      shap_value: float = Field(default=0.0, description="SHAP의 절대 기여 가중치")
      explanation_text: Optional[str] = Field(default=None, description="LLM이 번역한 자연어 해설 (REQ-FUNC-006)")

      forecast: Forecast = Relationship(back_populates="factors")
  ```

### 2. DTO 구조
- [ ] `app/schemas/forecast_schema.py`:
  - `FactorRead`: XAI 차트 시각화(DASH-002) 반환을 위한 단순 폼
  - `ForecastDetailRead`: 대시보드 메인 API가 응답할 전체 패키지(예측일, 예측수량, 신뢰도 포함 + 하위 Factors 리스트)

### 3. Repository 오퍼레이션
- [ ] `app/adapters/persistence/forecast_repository.py`:
  ```python
  # 엔드 투 엔드 저장: 예측 잡의 결과를 한번의 트랜잭션으로 커밋해 원자성 보장.
  async def save_forecast_with_factors(session: AsyncSession, forecast_data: Forecast, factors: List[ForecastFactor]) -> Forecast: ...
  
  # 대시보드 시각화 인출
  async def get_forecasts_by_date_range(session: AsyncSession, tenant_id: UUID, start_date: date, end_date: date) -> List[Forecast]: ...
  ```

## 🧪 Acceptance Criteria (BDD/GWT)

**Scenario 1: 원자성 및 부모 자식 영구 관계**
- Given: 엔진이 1개의 `Forecast` 와 3개의 `ForecastFactor` (Top 3 요인) 예측을 도출함.
- When: 레포지토리 단독 세션에서 이를 DB에 `save_forecast_with_factors` 로 쓰기(Commit) 기록.
- Then: DB 내 예측은 부모 자식의 외래키가 올바로 맺어지며, 추후 부모 예측 1개를 불러올때 factors 어트리뷰트 조인 호출을 통해 하위 인과 변동 3가지를 누락 없이 추적할 수 있다.

**Scenario 2: REQ-NF-026 기반 교차 노출 차단**
- Given: Tenant A가 테넌트 B의 `forecast_id` 를 알아내어 조회를 시도함.
- When: 조회 시 미들웨어 또는 레포지토리에서 `tenant_id` WHERE 질의 조건을 검.
- Then: 항상 None 인스턴스가 반환된다.

## ⚙️ Technical Constraints
- DTO 설계시 `confidence_level`은 % 출력일지 아니면 비율(0.XX) 상태로 전달할지를 명확한 주석과 스키마 validation으로 표기한다.
- `Predicted_qty`는 부동소수 결과이므로 프론트엔드에서 반올림 하거나, 서빙 전에 Integer 라운딩 오퍼레이션을 염두에 둔다.

## 🏁 Definition of Done (DoD)
- [ ] Forecast, ForecastFactor 두 모델이 연관(`Relationship`) 과 Cascade로 생성 완료?
- [ ] `tenant_id`, `product_id`를 의존하는 인덱스 외래키 구성 완료?
- [ ] Repository 에 Join 쿼리를 포함한 Read 함수가 구현되었나?
- [ ] Alembic 마이그레이션 적용 및 `confidence_level` 및 `shap_value` Float 필드 스키마 생성 정상 확인?
