---
name: Feature Task
about: SRS 기반의 구체적인 개발 태스크 명세
title: "[ETL] ETL-001: ETL 파이프라인 오케스트레이터 및 DAG 체계 확립"
labels: 'etl, backend, pipeline, priority:critical, phase:1'
assignees: ''
type: task
tags: [task, backend_core]
created: 2026-04-22
---
## 🎯 Summary
- 기능명: [ETL-001] ETL 파이프라인 오케스트레이터 및 배치 제어
- 목적: `INFRA-003` (APScheduler 인프라) 위에서 동작하는 논리적인 파이프라인 레이어를 구축한다. 다수의 수집 Worker들(기상, 트렌드, 주문 등)과 처리 Worker들(예측, 역산) 간의 순서(DAG: Directed Acyclic Graph 유사 동작)를 제어하고, 매일 정해진 스케줄 흐름대로 데이터를 밀어 넣게 조율한다.

## 🔗 References (Spec & Context)
- SRS 요구사항: [`SRS-V1.0.md#§4.1.4`](file:///e:/workspace/SRS-from-PRD/SRS-V1.0.md) — REQ-FUNC-027 (ETL 자동화 제어)
- SRS 아키텍처: [`SRS-V1.0.md#§3.6`](file:///e:/workspace/SRS-from-PRD/SRS-V1.0.md) — Batch & Scheduler 파트
- 의존성: C-TEC-008 (Airflow 대비 가벼운 APScheduler 사용에 대한 보완책)

## ✅ Task Breakdown (실행 계획)

### 1. Job Orchestrator 모듈 구성
- [ ] `app/core/etl_orchestrator.py` 작성:
  ```python
  from apscheduler.schedulers.asyncio import AsyncIOScheduler
  import logging
  from typing import Callable, List
  
  logger = logging.getLogger(__name__)

  class ETLOrchestrator:
      def __init__(self, scheduler: AsyncIOScheduler):
          self.scheduler = scheduler

      def register_job(self, 
                       job_func: Callable, 
                       job_id: str, 
                       cron_hour: int, 
                       cron_minute: int,
                       depends_on: List[str] = None):
          """
          개별 워커 등록 함수.
          Airflow의 bitshift(>>) 문법을 간소화하여 depends_on 파라미터로
          간단한 종속성 체인을 모방 (단순 시간차 배치 또는 완료 콜백 방식 사용)
          """
          self.scheduler.add_job(
              job_func,
              'cron',
              hour=cron_hour,
              minute=cron_minute,
              id=job_id,
              replace_existing=True,
              misfire_grace_time=3600
          )
          logger.info(f"ETL Job [{job_id}] scheduled at {cron_hour:02d}:{cron_minute:02d}")
  ```

### 2. Main ETL 스케줄 체인(DAG) 등록
- [ ] `app/api/lifespan.py` 또는 `main.py`에 전체 스케줄러 등록 규격화.
  ```python
  def setup_etl_pipelines(scheduler):
      orchestrator = ETLOrchestrator(scheduler)
      # 1. 외부 API 수집 파이프라인 (새벽)
      orchestrator.register_job(weather_worker_run, "f1_weather", 6, 0)
      orchestrator.register_job(trend_worker_run, "f1_trend", 7, 0)
      
      # 2. 내부 쇼핑몰 수집 (아침 출근 전)
      orchestrator.register_job(order_worker_run, "f2_orders", 8, 0)
      orchestrator.register_job(inventory_worker_run, "f2_inventory", 8, 30)

      # 3. 예측 프레딕션 파이프라인 (재고 결산 직후)
      # orchestrator.register_job(forecast_trigger, "f1_forecast", 9, 0) # 향후 개발

      # 4. 알림 톡 발송 파이프라인 (오후 근무자 통보 시점)
      orchestrator.register_job(alimtalk_worker_run, "f3_alimtalk", 16, 0)
  ```

### 3. 상태 추적용 Logging 체계 구축
- [ ] `DB-010` 의 `AuditLog` 에 "ETL_SUCCESS" / "ETL_FAIL" 레코드를 떨구는 유틸리티 데코레이터 작성. 

## 🧪 Acceptance Criteria (BDD/GWT)

**Scenario 1: 스케줄러 자동 등록 확인**
- Given: `setup_etl_pipelines` 가 `lifespan` 에서 주입된 상황.
- When: 서버가 가동되고 `/api/v1/jobs` 등 스케줄러 상태 확인 API 를 호출함.
- Then: 기상, 트렌드, 주문, 알림톡 의 각 ID 및 Cron 타임테이블이 DB Jobstore에 정상 삽입/조회 되는 것을 확인한다.

**Scenario 2: Job 데코레이터(로깅) 추적**
- Given: 6시 0분에 날씨 수집 Job이 스케줄에 따라 자동 트리거됨.
- When: `weather_worker_run` 내부 처리가 모두 완료(Return True) 됨.
- Then: DB 내 `AUDIT_LOG` 테이블에 `action="ETL_SUCCESS", entity_type="JOB", entity_id="f1_weather"` 가 즉각 타임스탬프와 함께 삽입된다.

## ⚙️ Technical Constraints
- 순수 `cron` 기반 트리거이므로, A가 끝나야 B가 작동하는 의존성(`depends_on`)의 경우 MVP 단계에서는 시차(Time gap)를 두는 방식(예: 8:00, 8:30)으로 회피할 수 있다. 필요시엔 콜백이나 DB 플래그 상태 머신을 고려한다.

## 🏁 Definition of Done (DoD)
- [ ] `ETLOrchestrator` 추상화 컨트롤러 작성 완료?
- [ ] 총 4~5개의 기준 스케줄 워커를 선언하는 Setup 함수 작성?
- [ ] 각 Worker 의 성공 여부를 `DB-010` 에 적재하는 데코레이터 구성?
