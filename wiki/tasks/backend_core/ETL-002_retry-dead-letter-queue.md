---
name: Feature Task
about: SRS 기반의 구체적인 개발 태스크 명세
title: "[ETL] ETL-002: Retry 전략 및 Dead Letter Queue 처리"
labels: 'etl, backend, pipeline, priority:high, phase:1'
assignees: ''
type: task
tags: [task, backend_core]
created: 2026-04-22
---
## 🎯 Summary
- 기능명: [ETL-002] Worker 실패 시 재시도 체계 및 DLQ 구현
- 목적: 외부 API의 일시적 장애나 데이터 DB 격리 충돌 등으로 워커가 에러를 뱉었을 때, 시스템을 복구하기 위해 간격 조율형(Exponential Backoff) 재시도를 3회 수행한다. 완전 실패 시 Slack 통보 및 Dead Letter Queue (DLQ 로그) 로 빠지게 하여 후속 처리를 돕는다. (REQ-NF-011 준수)

## 🔗 References (Spec & Context)
- SRS 요구사항: [`SRS-V1.0.md#§4.2.2`](raw/assets/SRS-V1.0.md) — REQ-NF-011 (ETL 재시도 정책 및 파이프라인 신뢰성)
- 연관 태스크: `ADAPT-006` (Slack 훅), `DB-010` (Audit 로그)

## ✅ Task Breakdown (실행 계획)

### 1. Exponential Backoff 재시도 데코레이터
- [ ] `app/core/retry_handler.py` 구성:
  ```python
  import asyncio
  import logging
  from functools import wraps
  from app.adapters.external.slack_notifier import SlackNotifierAdapter

  logger = logging.getLogger(__name__)

  def with_etl_retry(max_retries: int = 3, base_delay_sec: int = 30):
      def decorator(func):
          @wraps(func)
          async def wrapper(*args, **kwargs):
              attempts = 0
              current_delay = base_delay_sec
              
              while attempts < max_retries:
                  try:
                      return await func(*args, **kwargs)
                  except Exception as e:
                      attempts += 1
                      logger.warning(f"[ETL Retry] {func.__name__} failed (Attempt {attempts}/{max_retries}): {str(e)}")
                      
                      if attempts == max_retries:
                          # 최종 실패 처리 (DLQ 훅)
                          await handle_dead_letter(func.__name__, str(e))
                          raise e
                          
                      await asyncio.sleep(current_delay)
                      current_delay *= 2  # Exponential Backoff (30s -> 60s -> 120s)
          return wrapper
      return decorator
  ```

### 2. Dead Letter & Slack 통보 핸들러 구현
- [ ] `handle_dead_letter` 비동기 함수 구현:
  ```python
  async def handle_dead_letter(worker_name: str, error_msg: str):
      # 1. 슬랙 통보 (ADAPT-006)
      slack = SlackNotifierAdapter()
      await slack.send_alert(
          title=f"ETL Worker Failed: {worker_name}", 
          message=f"최대 재시도 횟수에 도달했습니다.\nError: {error_msg}"
      )
      
      # 2. AUDIT LOG 강제 기록 (DB-010 접근)
      # ... DB Session 획득 후 entity_type="DLQ", action="FAILED_ETL" 보관
  ```

## 🧪 Acceptance Criteria (BDD/GWT)

**Scenario 1: 일시적 네트워크 장애 방어**
- Given: `weather_worker_run` 함수에 `@with_etl_retry(max_retries=3)` 이 감싸져 있음.
- When: 1회차 실행 시 `ReadTimeout` 에러 발생, 2회차 실행(30초 후) 시 정상 동작.
- Then: 전체 스케줄러 Job이 크래시 나지 않고, 데코레이터 내부에서 1회 실패 캡처 후 30초 대기(asyncio.sleep), 이어진 재호출에서 값 반환으로 정상 수집 트랜잭션이 종료된다.

**Scenario 2: 최종 실패 (DLQ 큐잉 및 경보)**
- Given: 외부 제공사 서버(카페24 정기점검)가 완전히 다운되어 3회 내내 접속이 불가능함.
- When: 1차(실패) -> 대기(30s) -> 2차(실패) -> 대기(60s) -> 3차(실패) 가 발생.
- Then: `handle_dead_letter` 가 트리거 되며, 관리자 Slack 의 `#alert-pipeline` 채널에 재시도 초과 에러 알림이 날아가고 로직이 종료된다.

## ⚙️ Technical Constraints
- 백오프 슬립(`asyncio.sleep`) 도중 이벤트 루프가 멈추지 않아야(Block) FastAPI 의 타 API 서비스 응답에 지장을 주지 않는다. 절대로 `time.sleep()` 을 사용하지 말 것.

## 🏁 Definition of Done (DoD)
- [ ] `with_etl_retry` 데코레이터가 Exponential 로직을 띄며 작성되었나?
- [ ] 재시도 한도 초과 시 Slack 으로 경보가 포워딩 되는가?
- [ ] 데코레이터가 모든 개별 Worker 들(F1, F2 등) 상단에 적용 가능한 구조인가?
