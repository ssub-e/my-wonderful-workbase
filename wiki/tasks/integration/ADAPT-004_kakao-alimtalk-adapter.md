---
name: Feature Task
about: SRS 기반의 구체적인 개발 태스크 명세
title: "[Adapter] ADAPT-004: 카카오 알림톡(비즈메시징) API 어댑터"
labels: 'adapter, external-api, phase:1'
assignees: ''
type: task
tags: [task, integration]
created: 2026-04-22
---
## 🎯 Summary
- 기능명: [ADAPT-004] 카카오 알림톡 API (Bizm 또는 알리고 등 벤더 연동) 어댑터
- 목적: 매일 16시(또는 역산 완료 직후) 인력 배치 담당자에게 내일 필요 인원을 Push하는 비즈니스 액션(REQ-FUNC-022)을 외부에 전송하기 위한 메시지 샌더 프로토콜을 구축한다.

## 🔗 References (Spec & Context)
- 알림 전송 요구: [`SRS-V1.0.md#§4.1.3`](raw/assets/SRS-V1.0.md) — REQ-FUNC-022 매일 16:00 카카오톡 알림톡 발송
- 오류 전송: REQ-FUNC-023 알림톡 실패 시 휴대폰 SMS 대체 전환 정책(Fallback)

## ✅ Task Breakdown (실행 계획)

### 1. HTTP Alimtalk 통신 클라이언트 구축
- [ ] `app/adapters/external/kakao_notifier.py`
  ```python
  import httpx
  from typing import Optional

  class KakaoAlimtalkAdapter:
      def __init__(self, api_key: str, sender_key: str):
          self.api_key = api_key
          self.sender_key = sender_key
          self.base_url = "https://kakaoapi.aligo.in/v2/alimtalk" # 예시 벤더
          
      async def send_workforce_plan(self, phone_number: str, center_name: str, recommended_qty: int, date: str) -> bool:
          """템플릿 매핑 후 HTTP 전송, 성공 여부만 부울로 리턴 (단순화)"""
          pass
  ```

### 2. SMS Fallback 로직 결합 (REQ-FUNC-023 준수)
- [ ] 어댑터 내부에 구조적 Fallback 함수 추가: `async def send_sms_fallback(self, phone_number, text) -> bool`
- [ ] `send_workforce_plan` 메서드 내에서 카카오 전송 에러(`is_error == True`) 파악 시 자동적으로 `send_sms_fallback`을 호출하도록 설계하여 성공률을 높인다.

## 🧪 Acceptance Criteria (BDD/GWT)

**Scenario 1: 알림톡 정상 전송**
- Given: 대상 연락처와 `{"center": "용인 센터", "qty": 15}` 등의 발송 파라미터가 부여됨.
- When: 어댑터에 쏘기 요청 완료
- Then: 벤더사 API에서 200 OK 리턴 및 `send_workforce_plan` 함수가 `True` (성공) 부울을 리턴하여 호출부 기록(DB-009 이력)을 돕는다.

**Scenario 2: 수신자 카카오톡 미설치 등 실패 시 SMS Fallback 작동**
- Given: 카카오 메시지 전송 패킷이 벤더사에서 `1001: 카카오 연동 에러 (또는 미가입자)` 상태 코드로 리턴됨.
- When: 어댑터 로직 단
- Then: 이를 Catch 하여 SMS 대체 API 로 재전송하며, 이 Fallback 까지 완료 후 최종적으로 성공(`True` + 채널 식별자 'sms') 형태로 기록을 뱉어주어 시스템 무결성을 지킨다.

## ⚙️ Technical Constraints
- 카카오 알림톡은 사전 승인된 템플릿(코드)만을 통해 발송해야 한다. 템플릿 코드는 환결변수에 주입하고, `center_name`, `qty` 등 가변 변수만 `#{}` 규격으로 조립해서 치환한다.

## 🏁 Definition of Done (DoD)
- [ ] 통신 어댑터 구조 및 알림톡/SMS 벤더용 엔드포인트 세팅 완료?
- [ ] 비즈 로직 상 SMS Fallback 자동 전환 구조가 작성되었는가 (REQ-FUNC-023 통과)?
