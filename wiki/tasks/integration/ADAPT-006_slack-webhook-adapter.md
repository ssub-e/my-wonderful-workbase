---
name: Feature Task
about: SRS 기반의 구체적인 개발 태스크 명세
title: "[Adapter] ADAPT-006: Slack Webhook 이벤트 알림 어댑터"
labels: 'adapter, external-api, phase:0, priority:low'
assignees: ''
type: task
tags: [task, integration]
created: 2026-04-22
---
## 🎯 Summary
- 기능명: [ADAPT-006] Slack Webhook 기반 시스템 장애 알람 어댑터 구축
- 목적: 개발-운영(DevOps) 모니터링 목적으로, ETL 스케줄러 배치 실패, 429 무한루프, 서버 Exception 등 크리티컬한 장애 발생 시 개발팀의 `#alert-pipeline` 채널에 즉각적인 알림(메시지 전송)을 보내 골든타임을 확보한다(REQ-NF-022 정책 충족).

## 🔗 References (Spec & Context)
- SRS 모니터링: [`SRS-V1.0.md#§4.2.5`](raw/assets/SRS-V1.0.md) — REQ-NF-022(파이프라인 모니터링 경보를 Slack 등으로 알림)
- 의존성: `INFRA-003`(스케줄러 실패 통보 액션)

## ✅ Task Breakdown (실행 계획)

### 1. Webhook Sender 클래스 확립
- [ ] `app/adapters/external/slack_notifier.py` 세팅:
  ```python
  import httpx
  from app.core.config import settings

  class SlackNotifierAdapter:
      def __init__(self):
          self.webhook_url = settings.SLACK_WEBHOOK_URL
          
      async def send_alert(self, title: str, message: str, level: str = "ERROR") -> None:
          """
          level 에 따라 이모지(🚨, ⚠️, ℹ️) 를 붙여서 JSON 마크다운 포맷 구성 후
          Incoming Webhook 엔드포인트에 httpx 로 Fire-And-Forget 전송.
          """
          if not self.webhook_url:
              return # 설정 미비 시 패스 (개발환경 방해 방지)
              
          payload = {
              "blocks": [
                  {
                      "type": "section",
                      "text": {"type": "mrkdwn", "text": f"*{level}*: {title}\n```{message}```"}
                  }
              ]
          }
          async with httpx.AsyncClient() as client:
              # Fire and forget (에러 나도 시스템에 영향 X)
              try:
                  await client.post(self.webhook_url, json=payload, timeout=3.0)
              except Exception:
                  pass 
  ```

### 2. 연동 지점 설정 (Application Layer)
- [ ] `app/main.py` 의 기본 `Exception Handler` 나 `INFRA-003`의 Job 미스파이어 섹션에 이 `send_alert` 발송 코드 임베딩 준비.

## 🧪 Acceptance Criteria (BDD/GWT)

**Scenario 1: Exception 발생 시 발송 동작**
- Given: `SLACK_WEBHOOK_URL` 이 지정된 운용 모드 환경
- When: 어댑터에 텍스트를 담아 호출
- Then: Slack 채널에 블럭 기반 포매팅(``` code ``` 영역 포함) 메시지가 도착하며 업무망 내 즉시 인지 수단으로 작동한다.

**Scenario 2: Fail-Safe 및 타 기능 영향도 0 보장**
- Given: 잘못된 Webhook URL 이 기입되거나 사내 인터넷 차단이 걸린 서버망
- When: 어플리케이션 에러 로깅 과정 중 Slack을 쏘려 시도함
- Then: `httpx` 타임아웃이나 예외가 슬랙 어댑터 내부의 `except Exception: pass` 로 잡아먹히고, 클라이언트에게 내려가는 원래 API 응답이나 백엔드 처리(메인 DB 쓰기 등)를 저해(Crash)시키지 않는 구조다 (완벽 고립).

## ⚙️ Technical Constraints
- Fire & Forget 이 중요. 슬랙 서버 응답 지연이 우리 서버 체류 시간을 잡아먹게 둬선 안됨 (`timeout=3.0` 등 짧게 세팅 강제).

## 🏁 Definition of Done (DoD)
- [ ] Fire-and-forget 원칙이 지켜진 `SlackNotifierAdapter` 가 작성?
- [ ] 환경 변수(`SLACK_WEBHOOK_URL`) 스위칭으로 비활성 처리(if not) 가 가능한가?
