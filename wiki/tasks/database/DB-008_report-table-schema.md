---
name: Feature Task
about: SRS 기반의 구체적인 개발 태스크 명세
title: "[DB] DB-008: REPORT 보고서 이력 테이블 스키마 + 레파지토리"
labels: 'database, backend, priority:low, phase:1'
assignees: ''
type: task
tags: [task, database]
created: 2026-04-22
---
## 🎯 Summary
- 기능명: [DB-008] REPORT 테이블 스키마 및 레파지토리 작업
- 목적: "XAI 원클릭 리포트 추출" 기능 (REQ-FUNC-007)을 통해 PDF 형식 등으로 생성된 최종 파일의 경로 및 이력을 영구 보존. 경영진 제출 증빙, 과거 품의서 재열람, 그리고 감사 KPI (발주 반려 여부) 추적을 위한 파일 메타데이터를 저장한다.

## 🔗 References (Spec & Context)
- SRS 모델: [`SRS-V1.0.md#§6.2`](file:///e:/workspace/SRS-from-PRD/SRS-V1.0.md) — REPORT
- 관련 스펙: [`SRS-V1.0.md#§4.1.1`](file:///e:/workspace/SRS-from-PRD/SRS-V1.0.md) — REQ-FUNC-007 PDF 자동 생성
- 스토리지: C-TEC-011 (Supabase Storage / S3 호환 Object Storage)

## ✅ Task Breakdown (실행 계획)

### 1. SQLModel 형태 스키마 개발
- [ ] `app/domain/report/models.py` 구성:
  ```python
  class Report(SQLModel, table=True):
      __tablename__ = "reports"
      
      id: uuid.UUID = Field(default_factory=uuid.uuid4, primary_key=True)
      tenant_id: uuid.UUID = Field(foreign_key="tenants.id", index=True, nullable=False)
      forecast_id: Optional[uuid.UUID] = Field(foreign_key="forecasts.id", nullable=True) # 어떤 예측값을 기반으로 파생된 문서인지
      type: str = Field(default="pdf", max_length=20, description="pdf / excel 등 문서 타입")
      file_path: str = Field(nullable=False, description="S3 Key 또는 Storage Object 접근 URL 경로")
      status: str = Field(default="generated", max_length=30, description="processing / generated / failed / validated")
      generated_at: datetime = Field(default_factory=datetime.utcnow)
      
      # (Optional) REQ-NF-031 무수정 채택 척도 등을 위한 플래그
      is_edited_by_human: bool = Field(default=False, description="다운로드 후 수동 수정 반영 여부 추적(KPI)")
  ```

### 2. Output DTO 생성
- [ ] `app/schemas/report_schema.py`
  - `ReportRead`: 사용자 대시보드의 '과거 리포트 모아보기' 패널을 장식할 반환 구조체 (id, 파일 형태, 시각, 접근을 위한 URL 포함).

### 3. Repository 액션 구성
- [ ] `app/adapters/persistence/report_repository.py`
  ```python
  # 파일 생성 시작/도중 시 기록 남기기 (상태 관리 - processing)
  async def initiate_report(session: AsyncSession, tenant_id: UUID, forecast_id: UUID, file_path_placeholder: str) -> Report: ...
  
  # 파일 처리가 끝나고 S3 올라가면 (상태 갱신 - generated)
  async def update_report_status(session: AsyncSession, report_id: UUID, status: str, final_url: str) -> Report: ...
  
  # 페이지네이션 기반 조회
  async def get_reports_by_tenant(session: AsyncSession, tenant_id: UUID, skip: int = 0, limit: int = 20) -> List[Report]: ...
  ```

### 4. 마이그레이션 생성
- [ ] `alembic` 마스터를 이용한 테이블 Create 구문 자동산출

## 🧪 Acceptance Criteria (BDD/GWT)

**Scenario 1: S3 업로드 파이프라인 대응 스키마 수정**
- Given: 비동기 워커가 백그라운드 모델로 PDF 생성을 트리거함.
- When: 최초에 `initiate_report` 를 통해 status='processing' 인 메타 트랜잭션이 DB에 박힘
- Then: 파일 제네레이션 후 S3 업로드가 완료되면 `update_report_status`에 의해 상태가 `generated`로 업데이트 되며 사용자는 유효한 `file_path` 접근 링크를 얻는다.

**Scenario 2: 파일 이터레이터 관리**
- Given: Tenant ID로 특정 회사의 과거 전체 기록 호출을 요청 (`get_reports_by_tenant`)
- When: 2달전에 생성된 항목들을 Pagination 쿼리로 날림
- Then: 조건 필터링에 맞아 누락본 없이 오름/내림차순 정렬된 20개 내역을 즉각 표출한다.

## ⚙️ Technical Constraints
- `file_path` 값: 애플리케이션 코어는 S3 도메인을 하드코딩 하지 않고, 환경변수에 지정된 버킷의 Resource Path 값만을 기입한다. 클라이언트는 이를 조합하여 다운로드를 실행한다.

## 🏁 Definition of Done (DoD)
- [ ] Report SQLModel 모델이 텍스트로 잘 반영되었는가?
- [ ] Forecast 부모 관계가 FK로 맺어져 고아 문서 발생을 막고 있는가?
- [ ] 비동기 생성 시나리오를 감안한 Repository 상태 변경 함수 존재를 확인?
