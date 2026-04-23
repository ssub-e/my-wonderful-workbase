---
name: Feature Task
about: SRS 기반의 구체적인 개발 태스크 명세
title: "[Monitor] MONITOR-001: FastAPI 전역 5XX 에러 Slack 알림 미들웨어"
labels: 'backend, monitor, error-handling, phase:6, priority:high'
assignees: ''
type: task
tags: [task, monitoring]
created: 2026-04-22
---
## 🎯 Summary
- 기능명: [MONITOR-001] 예측 불가능한 서버 다운/크래시(500 HTTP Exception) 발생 시 Slack Webhook 전송 체계
- 목적: 고객이 화면에서 먼저 "서버가 안되는데요?" 라며 전화를 걸어오기 전에, 백엔드 엔진이 죽는 즉시 텔레그렘/슬랙(Slack) 채널로 Call Stack과 URL을 발송하여 무중단 운영 체계의 조기 진압망을 구성한다.

## 🔗 References (Spec & Context)
- 기반 어댑터: `ADAPT-006` (Slack Webhook 어댑터)
- 대상 계층: 인바운드 트래픽 최전선 (FastAPI Middleware 또는 Exception Handler)

## ✅ Task Breakdown (실행 계획)

### 1. 전역 Exception Handler 등록
- [ ] `app/api/errors/handlers.py`:
  ```python
  import traceback
  from fastapi import Request
  from fastapi.responses import JSONResponse
  from app.adapters.slack_adapter import send_slack_alert
  from app.core.config import settings

  async def global_500_exception_handler(request: Request, exc: Exception):
      """
      예상치 못한 모든 500 계열 서버 로직 에러를 낚아채서 슬랙으로 던짐
      """
      # 1. 스택 트레이스 문자열 파싱
      error_msg = f"[{request.method}] {request.url.path} ➡️ {str(exc)}"
      tb_str = "".join(traceback.format_exception(type(exc), exc, exc.__traceback__))
      
      # 2. Slack 발송 포맷 조립 (환경이 PROD 일때만 발송)
      if settings.ENV == "production":
          alert_text = f"🚨 *CRITICAL ERROR* 🚨\n*Endpoint*: {error_msg}\n*Traceback*:\n```\n{tb_str[:1500]}\n```" # 길이 너무 길면 잘림 방지
          await send_slack_alert(alert_text)
      
      # 3. 우아한 HTTP 반환 (Call Stack 자체를 고객에게 주는건 보안(Path Traversal) 공격 위협이 있음)
      return JSONResponse(
          status_code=500,
          content={"message": "서버 내부 오류가 발생했습니다. 관리자에게 리포트가 발송되었습니다.", "error": str(exc)}
      )
  ```

### 2. Main App 연동
- [ ] `app/main.py`:
  ```python
  from app.api.errors.handlers import global_500_exception_handler

  app.add_exception_handler(Exception, global_500_exception_handler)
  ```

## 🧪 Acceptance Criteria (BDD/GWT)

**Scenario 1: 예상된 에러(400) vs 예상치 못한 에러(500) 분기 통제**
- Given: 비밀번호를 틀려 `HTTPException(status_code=401)` 이 떨어지거나 데이터가 조회가 안돼 404가 떨어짐 
- When: 위 Handler가 동작
- Then: 이 핸들러는 Base `Exception` 중에서도 FastAPI가 제어하는 `HTTPException`(커스텀 4xx) 은 낚아채지 않고 통과시키며, 오직 진짜 버그에 의한 `KeyError`나 `DatabaseError` 등 500계열 크래시만 슬랙으로 전파해 채널의 피로도를 방어한다.

## ⚙️ Technical Constraints
- 알림이 터지는 것을 막기 위해 `tb_str` 길이는 1000~1500자로 잘라서(`[:1500]`) 슬랙 Webhook Block Kit의 사이즈 리밋을 우회해야 한다.

## 🏁 Definition of Done (DoD)
- [ ] `Exception` 을 넓게 낚아채는 전역 핸들러 구성 완료?
- [ ] Production 환경 분기(`ENV=='production'`) 제어 적용 유무?
