---
name: Feature Task
about: SRS 기반의 구체적인 개발 태스크 명세
title: "[DB] DB-006: 트렌드 데이터(TREND_DATA) 테이블 스키마 + 레파지토리"
labels: 'database, backend, priority:medium, phase:1'
assignees: ''
type: task
tags: [task, database]
created: 2026-04-22
---
## 🎯 Summary
- 기능명: [DB-006] TREND_DATA 스키마 및 레파지토리 셋업
- 목적: 네이버 DataLab 및 소셜 미디어 지표에서 도출된 검색량 지수 및 키워드 트렌드 추이를 저장. 날씨(WEATHER)와 마찬가지로 화주사(Tenant) 의존성 없이 카테고리/키워드별로 **글로벌 공용 데이터**로 보관되어 머신러닝 예측 피처로 작동한다.

## 🔗 References (Spec & Context)
- SRS 정보 보관: [`SRS-V1.0.md#§6.2`](raw/assets/SRS-V1.0.md) — TREND_DATA 엔터티
- 예측 수집 요구사항: [`SRS-V1.0.md#§4.1.1`](raw/assets/SRS-V1.0.md) — REQ-FUNC-002 네이버 DataLab 자동 수집 연동

## ✅ Task Breakdown (실행 계획)

### 1. SQLModel 스키마 생성
- [ ] `app/domain/forecast/models.py` (기상 데이터 모듈의 이어 쓰기 권장):
  ```python
  class TrendData(SQLModel, table=True):
      __tablename__ = "trend_data"
      
      id: uuid.UUID = Field(default_factory=uuid.uuid4, primary_key=True)
      date: date = Field(index=True, nullable=False)
      keyword: str = Field(index=True, max_length=200, nullable=False, description="마스크, 수분크림 등 추적 대상")
      search_volume: float = Field(default=0.0, description="네이버 DataLab 등에서 산출한 검색량 변동 지수(0~100)")
      source: str = Field(default="naver_datalab", max_length=50) # 데이터의 출처
  ```

### 2. DTO 구성
- [ ] `app/schemas/forecast_schema.py`:
  - `TrendDataCreate`: JSON 반환 API의 내용을 파싱해 담을 구조체
  - `TrendRead`: 추후 모델에 Input할 객체 반환 폼.

### 3. Repository 오퍼레이션
- [ ] `app/adapters/persistence/trend_repository.py`:
  ```python
  # ETL 전용 대량 인서트(Upsert 권장)
  async def bulk_upsert_trends(session: AsyncSession, trends: List[TrendDataCreate]) -> None: ...
  
  # 특정 키워드의 특정 구간 검색량 추이 분석 인출 (모델링 Feature 추출용)
  async def get_trend_history(session: AsyncSession, keyword: str, start_date: date, end_date: date) -> List[TrendData]: ...
  ```

### 4. 고유키 설정 (Alembic)
- [ ] `(date, keyword, source)` 조합으로 복합 Unique Constraint 적용. 중복 수집 멱등성 보장.

## 🧪 Acceptance Criteria (BDD/GWT)

**Scenario 1: 트렌드 지표 날짜 기반 범위 조회 (ML Feed 용도)**
- Given: 과거 100일간의 "수분크림" 검색량 지수 내역이 DB에 적재됨 
- When: 시스템 백엔드가 `get_trend_history` 함수로 `start_date`부터 `end_date` 까지의 구간 데이터를 호출함
- Then: 조건에 맞춘 시계열 `TrendData` 객체 배열이 오름차순으로 즉각 반환된다.

**Scenario 2: 중복 데이터 삽입 (ETL 오류 보완)**
- Given: 1월 1일의 `keyword="마스크"` 에 대해 크롤링 봇이 지수 40.5 로 데이터를 저장 완료함.
- When: 봇이 1월 1일 "마스크" 지수를 50.0 으로 Update 성향을 띠며 인서트 요청을 보냄
- Then: Unique 제약에 의해 신규 로우는 만들어 지지 않고 (Upsert 대응인 경우 수치만 수정되며) 반환된다.

## ⚙️ Technical Constraints
- 추후 텍스트 분석에 대비해 `keyword` 필드는 단순 varchar 외에도 인덱싱을 지원하는 구조를 염두.
- 검색량 자체(Volume)은 부동소수점(`float`) 관리로 통일 (네이버 API가 지수 기반의 소수점을 뱉음).

## 🏁 Definition of Done (DoD)
- [ ] `TrendData` SQLModel 에 `(date, keyword, source)` 유니크 제약 포함 구성했는가?
- [ ] Repository에 시계열 기반 기간 조회 기능이 생성되었는가?
- [ ] Alembic Migration 적용으로 테이블 프로비저닝 확인?
