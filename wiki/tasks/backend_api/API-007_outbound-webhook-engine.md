---
name: Feature Task
about: SRS 기반의 구체적인 개발 태스크 명세
title: "[Feature] API-007: 시스템 이벤트 통보용 '아웃바운드(Outbound) 알림 웹훅' 엔진"
labels: 'feature, backend, api, priority:medium'
assignees: ''
type: task
tags: [task, backend_api]
created: 2026-04-22
---
## 🎯 Summary
- 기능명: [API-007] 외부 시스템 연동을 위한 아웃바운드 푸시 웹훅
- 목적: REQ-FUNC-015 확장. 우리 시스템 내에서 중요한 이벤트(예: 예측 생성 완료, 재고 위험 알람 발생)가 일어났을 때, 고객사의 외부 서드파티 시스템(예: 사내 메신저봇, ERP 이벤트 리스너)에 HTTP POST 요청으로 즉시 통보해 주는 기능을 구축한다.

## 🔗 References (Spec & Context)
> 💡 AI Agent & Dev Note: 작업 시작 전 아래 문서들을 반드시 먼저 Read/Evaluate 할 것.
- 반대 기능: `API-005` (Inbound Webhook)
- 보안 정책: `SEC-004` (민감 로그 기록)

## ✅ Task Breakdown (실행 계획)
- [ ] `WEBHOOK_SUBSCRIPTION` 테이블 설계: id, tenant_id, target_url, event_types(Enum Array), secret_token, is_active
- [ ] **Webhook Dispatcher** 구현:
  - 시스템 내부 이벤트 발생 시(Event-driven) 발행(Pub/Sub)
  - 등록된 구독자 URL로 비동기 HTTP 요청 송신 (`httpx` 활용)
  - 지수 백오프(Exponential Backoff) 기반 재시도 로직 적용
- [ ] **Payload Signing**: 보안을 위해 전송하는 JSON 바디를 `secret_token` 으로 HMAC-SHA256 서명하여 헤더에 포함
- [ ] **Webhook Log Viewer**: 고객사 관리자가 전송 성공/실패 여부를 확인하고 수동으로 재전송(`Retry`) 할 수 있는 관리 화면 API
- [ ] 엔드포인트: `POST /api/v1/admin/webhooks/subscriptions` (URL 등록)

## 🧪 Acceptance Criteria (BDD/GWT)
Scenario 1: 예측 완료 이벤트 통보 성공
- Given: 테넌트 A가 `FORECAST_COMPLETED` 이벤트를 특정 URL로 구독함
- When: 일일 AI 예측 워커가 수행을 완수함
- Then: 테넌트 A가 등록한 URL로 `forecast_id`, `status` 등이 포함된 JSON 페이로드가 즉시 전송되어야 한다.

Scenario 2: 타겟 서버 장애 시 재시도 수행
- Given: 고객사 수신 서버가 일시적인 500 에러를 반환함
- When: 웹훅 전송 시도
- Then: 시스템은 즉시 포기하지 않고, 1분, 5분, 15분 간격으로 총 3회 재시도를 수행한 뒤 최종 실패 시 로그에 기록한다.

## ⚙️ Technical & Non-Functional Constraints
- 성능: 웹훅 전송 작업이 메인 비즈니스 로직(예측 연산 등)의 성능에 영향을 주지 않도록 오직 백그라운드 워커에서 처리
- 보안: `X-Hub-Signature` 와 같은 표준 서명 헤더를 사용하여 수신측에서 요청의 무결성을 검증할 수 있게 함
- 가용성: 웹훅 타겟 서버가 의도적으로 무한 루프를 돌거나 응답하지 않을 경우를 대비해 `Timeout: 5s` 강제 적용

## 💻 Implementation Snippet (Outbound Dispatcher)
```python
import hmac
import hashlib

async def send_webhook(target_url: str, secret: str, payload: dict):
    """
    서명된 페이로드를 외부 URL로 전송
    """
    body = json.dumps(payload)
    signature = hmac.new(
        secret.encode(), 
        body.encode(), 
        hashlib.sha256
    ).hexdigest()
    
    async with httpx.AsyncClient() as client:
        try:
            response = await client.post(
                target_url,
                content=body,
                headers={"X-Webhook-Signature": signature, "Content-Type": "application/json"},
                timeout=5.0
            )
            return response.status_code == 200
        except Exception:
            return False
```

## 🏁 Definition of Done (DoD)
- [ ] 모든 Acceptance Criteria를 충족하는가?
- [ ] HMAC 서명 검증 샘플 코드가 고객사용 가이드에 포함되어 있는가?
- [ ] `WEBHOOK_LOG` 에 전송 결과(Status Code, Payload)가 정확히 저장되는가?

## 🚧 Dependencies & Blockers
- Depends on: `INFRA-003` (APScheduler for retries)
- Blocks: 없음
