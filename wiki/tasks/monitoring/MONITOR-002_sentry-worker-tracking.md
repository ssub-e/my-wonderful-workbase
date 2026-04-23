---
name: Feature Task
about: SRS 기반의 구체적인 개발 태스크 명세
title: "[Monitor] MONITOR-002: 데몬/워커 스케줄러(ETL) 백그라운드 추락 감지망 (Sentry)"
labels: 'backend, worker, monitor, phase:6, priority:medium'
assignees: ''
type: task
tags: [task, monitoring]
created: 2026-04-22
---
## 🎯 Summary
- 기능명: [MONITOR-002] 스케줄러 배치 모듈 연산 실패 추적기 연동
- 목적: API 호출과 달리, 새벽 2시에 백그라운드에서 조용히 도는 ETL(`F1`, `F2` 워커들)은 추락해도 즉각 표가 나지 않는다. 이 때문에 Sentry SDK를 워커 컨텍스트에 묶어, 데드락이나 연동사 API 타임아웃 발생 시 즉각 이슈 트래킹 대시보드에 티켓을 밀어넣는다.

## 🔗 References (Spec & Context)
- 타겟 모듈: `ETL-001` (APScheduler Job 데몬)
- 도입 검토 툴: Sentry (에러 로깅 스탠다드)

## ✅ Task Breakdown (실행 계획)

### 1. Sentry 의존성 초기화
- [ ] `npm(설치안함) / pip install sentry-sdk`
- [ ] `app/core/config.py`: `SENTRY_DSN` 환경 변수 속성 추가.

### 2. Lifespan 에 Sentry Init 매핑
- [ ] `app/main.py` 의 `lifespan` 구역:
  ```python
  import sentry_sdk
  from app.core.config import settings

  if settings.SENTRY_DSN:
      sentry_sdk.init(
          dsn=settings.SENTRY_DSN,
          traces_sample_rate=0.2, # 성능 추적 20%만
          environment=settings.ENV,
      )
  ```

### 3. Worker 데코레이터 내 Sentry 트래픽 바인딩
- [ ] `app/workers/decorators.py` (기존 `with_etl_retry` 수정):
  ```python
  import sentry_sdk
  
  def with_etl_retry(max_retries=3):
      def decorator(func):
          async def wrapper(*args, **kwargs):
              try:
                  return await func(*args, **kwargs)
              except Exception as e:
                  # 최종 실패 시 Sentry 로 즉각 리포팅
                  sentry_sdk.capture_exception(e)
                  # 데드 레터 등 후속 조치
                  raise e
          return wrapper
      return decorator
  ```

## 🧪 Acceptance Criteria (BDD/GWT)

**Scenario 1: Sentry Context 분리**
- Given: 네이버 트렌드 워커(`F1-W02`) 가 네이버 점검으로 계속 에러를 뿜음.
- When: `with_etl_retry` 내부 루프가 3회 동작 후 돌이킬 수 없음을 인지하고 `capture_exception(e)` 를 그림.
- Then: Sentry 대시보드에는 에러 코드망 함께 `Tag` 혹은 스택 체인 리스트를 통해 어떤 워커(함수명)가 죽었는지 명확하게 식별되며, 단순 API 500 에러와는 별도의 배치(Batch) 에러로 분류됨.

## ⚙️ Technical Constraints
- 성능 추적(Traces) 기능은 로드율에 무리가 가므로, 트랜잭션 샘플 레이트(`traces_sample_rate`)를 0.1~0.2 수준으로 낮춰놔야 서버 메모리에 병목을 유발시키지 않는다.

## 🏁 Definition of Done (DoD)
- [ ] Sentry DSN 설정 및 환경(Production/Dev) 분리 스니펫 적용?
- [ ] Worker 데코레이터에 Exception 캡처 트리거 작성 완료?
