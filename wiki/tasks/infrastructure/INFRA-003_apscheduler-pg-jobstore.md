---
name: Feature Task
about: SRS 기반의 구체적인 개발 태스크 명세
title: "[Infra] INFRA-003: APScheduler + PostgreSQL jobstore ETL 스케줄링 인프라"
labels: 'infra, backend, priority:high, phase:1'
assignees: ''
type: task
tags: [task, infrastructure]
created: 2026-04-22
---
## 🎯 Summary
- 기능명: [INFRA-003] APScheduler + PostgreSQL jobstore ETL 스케줄링 인프라
- 목적: 기상청/네이버/카페24 등 외부 API 데이터를 일 1회 이상 자동 수집하고, 매일 16:00 알림톡을 발송하는 **ETL 스케줄러의 기반 인프라**를 구축한다. APScheduler를 FastAPI에 내장하며, PostgreSQL jobstore로 스케줄을 영속화하여 서비스 재시작 시에도 스케줄이 유실되지 않도록 보장한다 (ADJ-05).

## 🔗 References (Spec & Context)
> 💡 AI Agent & Dev Note: 작업 시작 전 아래 문서를 반드시 먼저 Read/Evaluate 할 것.
- SRS 기술 스택: [`SRS-V1.0.md#§1.5.3`](file:///e:/workspace/SRS-from-PRD/SRS-V1.0.md) — C-TEC-008 APScheduler + PG jobstore 영속화
- SRS ETL 요구사항: [`SRS-V1.0.md#§4.1.4`](file:///e:/workspace/SRS-from-PRD/SRS-V1.0.md) — REQ-FUNC-027 ETL 자동화
- SRS 폴백 전략: [`SRS-V1.0.md#§3.1.1`](file:///e:/workspace/SRS-from-PRD/SRS-V1.0.md) — 외부 시스템 장애 시 임시 우회 전략
- SRS Reliability: [`SRS-V1.0.md#§4.2.2`](file:///e:/workspace/SRS-from-PRD/SRS-V1.0.md) — REQ-NF-011 ETL 재시도 정책
- SRS 시퀀스: [`SRS-V1.0.md#§6.3.1`](file:///e:/workspace/SRS-from-PRD/SRS-V1.0.md) — 데이터 수집 파이프라인 + Exception 처리 시퀀스
- SRS 알림 발송: [`SRS-V1.0.md#§3.4.3`](file:///e:/workspace/SRS-from-PRD/SRS-V1.0.md) — 적정 인원 역산 및 알림 흐름 시퀀스
- SRS Monitoring: [`SRS-V1.0.md#§4.2.5`](file:///e:/workspace/SRS-from-PRD/SRS-V1.0.md) — REQ-NF-022 ETL 파이프라인 모니터링
- SRS Traceability: [`SRS-V1.0.md#§5.1`](file:///e:/workspace/SRS-from-PRD/SRS-V1.0.md) — TC-027: ETL 자동 재시도 검증
- **관련 스케줄 대상 (등록할 Job 목록):**
  - 기상 데이터 수집: 매일 06:00 (REQ-FUNC-001)
  - 트렌드 데이터 수집: 매일 07:00 (REQ-FUNC-002)
  - 주문/재고 데이터 수집: 매일 08:00 + 연동 쇼핑몰별 (REQ-FUNC-013, 014)
  - 알림톡 발송: 매일 16:00 (REQ-FUNC-022)

## ✅ Task Breakdown (실행 계획)

### 1. APScheduler 기본 설정
- [ ] `apscheduler` 패키지 설치 (`APScheduler>=3.10`)
- [ ] `app/core/scheduler.py` — 스케줄러 싱글턴 모듈:
  ```python
  from apscheduler.schedulers.asyncio import AsyncIOScheduler
  from apscheduler.jobstores.sqlalchemy import SQLAlchemyJobStore
  from apscheduler.executors.pool import ThreadPoolExecutor, ProcessPoolExecutor
  
  jobstores = {
      'default': SQLAlchemyJobStore(url=settings.DATABASE_URL)
  }
  executors = {
      'default': ThreadPoolExecutor(20),
      'processpool': ProcessPoolExecutor(5)
  }
  job_defaults = {
      'coalesce': True,           # 밀린 실행 합치기
      'max_instances': 1,         # 동시 실행 방지
      'misfire_grace_time': 3600  # 1시간 이내 미스파이어 허용
  }
  
  scheduler = AsyncIOScheduler(
      jobstores=jobstores,
      executors=executors,
      job_defaults=job_defaults,
      timezone='Asia/Seoul'
  )
  ```

### 2. FastAPI Lifespan 통합
- [ ] `app/main.py` lifespan 이벤트에 스케줄러 시작/종료 연결:
  ```python
  @asynccontextmanager
  async def lifespan(app: FastAPI):
      scheduler.start()
      yield
      scheduler.shutdown()
  ```
- [ ] 서비스 재시작 시 PG jobstore에서 기존 스케줄 자동 복원 검증

### 3. 간격 조율형 재시도 (Exponential Backoff)
- [ ] `app/core/retry.py` — 재시도 유틸리티:
  ```python
  async def with_retry(func, max_retries=3, base_delay=30, max_delay=300):
      """
      간격 조율형 재시도 3회 (REQ-FUNC-027, REQ-NF-011)
      - 1차: 30초 후
      - 2차: 60초 후 (x2)
      - 3차: 120초 후 (x2)
      - 3회 실패 시 Slack 알림 발송
      """
  ```
- [ ] 재시도 횟수, 지연 시간, 최종 실패 시 콜백(Slack) 설정 가능
- [ ] 각 재시도 시도 및 결과를 AUDIT_LOG에 기록

### 4. Slack 알림 연동 (실패 통보)
- [ ] `app/adapters/external/slack_notifier.py` — Slack Webhook 어댑터:
  - [ ] ETL 실패 시 `#alert-pipeline` 채널로 알림 발송 (REQ-NF-022)
  - [ ] 알림 내용: Job 이름, 실패 시각, 에러 메시지, 재시도 횟수
  - [ ] 환경 변수: `SLACK_WEBHOOK_URL`

### 5. Job 등록 인터페이스 (스텁)
- [ ] `app/core/jobs.py` — 스케줄 Job 등록 함수:
  ```python
  def register_default_jobs(scheduler):
      """모든 기본 ETL/알림 스케줄 등록"""
      # 기상 데이터 수집 (매일 06:00 KST)
      scheduler.add_job(
          collect_weather_data,
          'cron', hour=6, minute=0,
          id='weather_collection',
          replace_existing=True,
          jobstore='default'
      )
      # 트렌드 데이터 수집 (매일 07:00 KST)
      # 주문/재고 수집 (매일 08:00 KST)
      # 알림톡 발송 (매일 16:00 KST) — REQ-FUNC-022
  ```
- [ ] 각 Job 함수는 빈 스텁으로 작성 (실제 로직은 F1-W01, F1-W02, F2-W03, F3-W03에서 구현)
- [ ] Job 등록 시 `replace_existing=True`로 중복 등록 방지

### 6. 스케줄러 관리 API (운영용)
- [ ] `app/api/v1/scheduler.py` — 관리자 전용 API (ADMIN 역할만):
  - [ ] `GET /api/v1/admin/scheduler/jobs` — 현재 등록된 전체 Job 목록 조회
  - [ ] `POST /api/v1/admin/scheduler/jobs/{job_id}/trigger` — 특정 Job 즉시 실행 (수동 트리거)
  - [ ] `GET /api/v1/admin/scheduler/jobs/{job_id}/history` — Job 실행 이력 조회

### 7. Job 실행 이력 테이블
- [ ] `JOB_EXECUTION_LOG` 테이블 생성 (마이그레이션):
  ```python
  class JobExecutionLog(SQLModel, table=True):
      __tablename__ = "job_execution_logs"
      
      id: uuid.UUID = Field(default_factory=uuid.uuid4, primary_key=True)
      job_id: str = Field(nullable=False, index=True)
      job_name: str = Field(nullable=False)
      status: str = Field(nullable=False)  # success, failed, retrying
      started_at: datetime
      finished_at: Optional[datetime]
      error_message: Optional[str]
      retry_count: int = Field(default=0)
      metadata: Optional[dict] = Field(default=None, sa_column=Column(JSON))
  ```

## 🧪 Acceptance Criteria (BDD/GWT)

**Scenario 1: 서비스 기동 시 스케줄러 자동 시작**
- Given: FastAPI 앱이 기동될 때
- When: lifespan startup 이벤트가 트리거됨
- Then: APScheduler가 시작되고, PG jobstore에 저장된 기존 Job들이 자동 복원되어 등록된다.

**Scenario 2: 서비스 재시작 시 스케줄 복원 (영속화 검증)**
- Given: 스케줄러에 4개 Job(기상, 트렌드, 주문, 알림톡)이 등록된 상태에서 서비스를 종료
- When: 서비스를 재시작
- Then: PG jobstore에서 4개 Job이 모두 복원되어 스케줄이 유실되지 않는다. (ADJ-05 핵심 검증)

**Scenario 3: ETL Job 실패 → 간격 조율형 재시도 3회**
- Given: 기상 수집 Job이 외부 API 장애로 실패할 때
- When: 1차 시도 실패
- Then: 30초 후 2차 → 60초 후 3차 → 120초 후 재시도. 3회 실패 시 Slack `#alert-pipeline` 알림 발송. 각 시도 결과가 JOB_EXECUTION_LOG에 기록된다.

**Scenario 4: 동일 Job 중복 실행 방지**
- Given: 기상 수집 Job이 실행 중인 상태 (max_instances=1)
- When: 같은 Job의 다음 스케줄 시간이 도래
- Then: 실행 중인 인스턴스가 완료될 때까지 새 실행이 대기하거나 합쳐진다 (coalesce=True).

**Scenario 5: 관리자 수동 트리거**
- Given: ADMIN 역할 사용자가 인증된 상태
- When: `POST /api/v1/admin/scheduler/jobs/weather_collection/trigger` 요청
- Then: 기상 수집 Job이 즉시 실행되고, 실행 결과가 JOB_EXECUTION_LOG에 기록된다.

**Scenario 6: 비관리자의 스케줄러 API 접근 차단**
- Given: MD 역할 사용자의 JWT
- When: `GET /api/v1/admin/scheduler/jobs` 요청
- Then: `403 Forbidden` 반환.

**Scenario 7: misfire 처리**
- Given: 서버 다운타임으로 인해 예정된 06:00 기상 수집이 미실행된 상태
- When: 06:30에 서버가 복구됨 (misfire_grace_time=3600 이내)
- Then: 밀린 Job이 즉시 1회 실행된다 (coalesce=True).

## ⚙️ Technical & Non-Functional Constraints
- **C-TEC-008 준수**: APScheduler (FastAPI 내장) + PostgreSQL jobstore 영속화 필수 (ADJ-05)
- **REQ-FUNC-027 준수**: ETL 수집→변환→적재 자동화, 간격 조율형 재시도 3회 + Slack 알림
- **REQ-NF-011 준수**: ETL 실패 시 간격 조율형 자동 재시도 3회 + Slack 알림
- **REQ-NF-022 준수**: ETL 모니터링 — DAG 실패 즉시 Slack 알림
- **Timezone**: `Asia/Seoul` (한국 시간 기준 스케줄링)
- **멱등성**: 모든 Job은 멱등적으로 설계 — 동일 날짜 재실행 시 데이터 중복 없음
- **Phase 2 전환 경로**: Airflow 전환 가능하도록 각 Job 함수를 순수 Python 함수로 분리 (C-TEC-008)
- **비동기**: AsyncIOScheduler 사용 — FastAPI의 이벤트 루프와 통합

## 🏁 Definition of Done (DoD)
- [ ] 모든 Acceptance Criteria(7개 시나리오)를 충족하는가?
- [ ] APScheduler가 FastAPI lifespan에 통합되어 자동 시작/종료되는가?
- [ ] PostgreSQL jobstore에 스케줄이 영속화되고, 재시작 시 복원되는가? (ADJ-05 핵심)
- [ ] 간격 조율형 재시도(30s → 60s → 120s)가 정상 동작하는가?
- [ ] 3회 실패 시 Slack 알림이 발송되는가?
- [ ] JOB_EXECUTION_LOG에 모든 실행 이력이 기록되는가?
- [ ] 관리자 전용 스케줄러 API가 RBAC으로 보호되는가?
- [ ] Job 함수 스텁 4개(기상, 트렌드, 주문, 알림톡)가 등록되어 있는가?
- [ ] 단위 테스트 + 재시도 시뮬레이션 테스트가 통과하는가?
- [ ] Timezone이 Asia/Seoul로 설정되어 한국 시간 기준으로 동작하는가?

## 🚧 Dependencies & Blockers
- **Depends on**:
  - INFRA-001 (PostgreSQL DB — jobstore 저장소)
  - ENV-006 (FastAPI 기본 구조 — lifespan 이벤트 프레임)
  - AUTH-001 (RBAC — 관리자 API 접근 제어)
- **Blocks**:
  - ETL-001 (ETL 오케스트레이터 — 이 스케줄러 위에 구현)
  - F1-W01 (기상 데이터 수집 — Job 함수 구현)
  - F1-W02 (트렌드 데이터 수집 — Job 함수 구현)
  - F2-W03 (주문 데이터 수집 — Job 함수 구현)
  - F2-W04 (재고 데이터 수집 — Job 함수 구현)
  - F3-W03 (16:00 알림톡 발송 — Job 함수 구현)
  - MON-002 (ETL 파이프라인 모니터링)
