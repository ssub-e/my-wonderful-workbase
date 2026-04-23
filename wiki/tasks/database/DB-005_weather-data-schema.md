---
name: Feature Task
about: SRS 기반의 구체적인 개발 태스크 명세
title: "[DB] DB-005: 기상 데이터(WEATHER_DATA) 테이블 스키마 + 레파지토리"
labels: 'database, backend, priority:medium, phase:1'
assignees: ''
type: task
tags: [task, database]
created: 2026-04-22
---
## 🎯 Summary
- 기능명: [DB-005] WEATHER_DATA 스키마 및 레파지토리 셋업
- 목적: 기상청 단기 예보로 수집된 전국 주요 권역별 날씨 데이터(강수량, 온도, 특보 등)를 일 단위로 격리 보관한다. 이 테이블은 특정 화주사(Tenant) 소속이 아닌 **글로벌 공용 데이터**로서, 예측 모델(LightGBM) 피처의 공통 입력 벡터로 활용된다.

## 🔗 References (Spec & Context)
- SRS 정보 보관: [`SRS-V1.0.md#§6.2`](file:///e:/workspace/SRS-from-PRD/SRS-V1.0.md) — WEATHER_DATA 엔터티
- 예측 활용 수요: [`SRS-V1.0.md#§4.1.1`](file:///e:/workspace/SRS-from-PRD/SRS-V1.0.md) — REQ-FUNC-001 기상청 자동 수집 연결

## ✅ Task Breakdown (실행 계획)

### 1. SQLModel 스키마 모델링
- [ ] `app/domain/forecast/models.py` 내 `WeatherData` 엔터티 추가:
  ```python
  from sqlmodel import SQLModel, Field
  import uuid
  from datetime import date
  from typing import Optional

  class WeatherData(SQLModel, table=True):
      __tablename__ = "weather_data"
      
      id: uuid.UUID = Field(default_factory=uuid.uuid4, primary_key=True)
      date: date = Field(index=True, nullable=False, description="예보/관측 대상 일자 기준")
      region: str = Field(index=True, max_length=100, nullable=False, description="행정 권역 혹은 물류센터 권역")
      temperature: float = Field(default=0.0)
      precipitation: float = Field(default=0.0, description="강수량(mm)")
      alert_level: Optional[str] = Field(default=None, description="주의보/경보 수준 텍스트")
  ```
- **ERD 제약 명심:** `WeatherData` 에는 `tenant_id` 필드가 없다. 공공 데이터이므로 교차 노출 문제가 적용되지 않는다.

### 2. DTO 구조 정의
- [ ] `app/schemas/forecast_schema.py`:
  - `WeatherDataCreate`: ETL 잡에서 인서트할 때 사용하는 모델
  - `WeatherRead`: 대시보드 참고 또는 API 로 응답시 사용

### 3. Repository (데이터 오퍼레이션)
- [ ] `app/adapters/persistence/weather_repository.py`:
  ```python
  # Upsert 로직: ETL 봇이 재실행되어도 동일(date, region) 쌍엔 업데이트(멱등성 보장)
  async def upsert_weather_data(session: AsyncSession, weather: WeatherDataCreate) -> WeatherData: ...
  
  # 예측 엔진을 위한 인출
  async def get_weather_by_date_and_region(session: AsyncSession, target_date: date, region: str) -> Optional[WeatherData]: ...
  ```

### 4. 고유 제약조건 (Unique Index)
- [ ] Alembic Migration 세팅 시, `(date, region)` 조합을 복합 유니크 제약(UniqueConstraint)으로 처리하여 한 권역의 하루에 데이터가 두 줄 이상 생성되지 않게 막는다.

## 🧪 Acceptance Criteria (BDD/GWT)

**Scenario 1: 복합 고유키 제약 기반 무결성 방어**
- Given: `date="2026-05-01"`, `region="seoul"` 인 날씨 데이터가 DB에 삽입 완료됨
- When: 스케줄러 중복 에러로 동일한 날짜/지역의 로우를 인서트하고자 함
- Then: DB단위의 Integrity 에러가 발생하여 이중 적재되지 않는다.

**Scenario 2: 예측 모델 대상 데이터 조회 속도(Index)**
- Given: 수년에 걸친 대량의 기상 데이터가 적재된 환경
- When: 특정 일자의 특정 지역 날씨(`where date=? and region=?`)를 조회
- Then: 해당 필드가 인덱스(B-Tree) 처리 되어 있으므로, 풀 테이블 스캔 없이 고속으로 객체가 리턴된다.

## ⚙️ Technical Constraints
- 공통 데이터이므로 Tenant 필터 의존성(`Auth` 미들웨어 등)에서 예외 처리되거나 필터 적용 전 단계에서 쿼리되어야 한다.

## 🏁 Definition of Done (DoD)
- [ ] `WeatherData` 스키마 및 `(date, region)` 유니크 제약 세팅 완료?
- [ ] Upsert 및 조회용 Repository 뼈대 함수 작성 완료?
- [ ] Alembic 생성 및 DB 적용 성공 여부 확인?
