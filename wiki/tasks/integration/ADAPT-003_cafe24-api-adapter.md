---
name: Feature Task
about: SRS 기반의 구체적인 개발 태스크 명세
title: "[Adapter] ADAPT-003: 카페24 REST API 연동 어댑터"
labels: 'adapter, external-api, phase:1, priority:critical'
assignees: ''
type: task
tags: [task, integration]
created: 2026-04-22
---
## 🎯 Summary
- 기능명: [ADAPT-003] 카페24 쇼핑몰 (Auth 및 주문/재고 조회) API 어댑터
- 목적: B2B 화주사의 핵심 원천 데이터인 "이전 주문"과 "현재 재고"를 긁어오기 위한 메인 커넥터를 생산한다. REQ-FUNC-015 규정에 따른 강력한 분당 API 호출 제한(Rate Limit) 방어 전략이 어댑터 본체 혹은 상위 프록시에 구현되어야 한다.

## 🔗 References (Spec & Context)
- SRS 요구사항: [`SRS-V1.0.md#§4.1.2`](file:///e:/workspace/SRS-from-PRD/SRS-V1.0.md) — REQ-FUNC-013(주문 수집), REQ-FUNC-014(재고 수집)
- 성능 제약: REQ-FUNC-015 — 카페24 분당 API 제한률 극복 설계(Redis 캐싱 등 연계)
- 헥사고날 룰 준수

## ✅ Task Breakdown (실행 계획)

### 1. OAuth 2.0 Base 컴포넌트 
- [ ] `app/adapters/external/cafe24/auth.py`
  - 테넌트의 DB에 저장된 `refresh_token` 을 재이용해 만료된 AccessToken을 실시간 갱신하는 헬퍼 함수 (`refresh_cafe24_token`).  *DB 의존성을 제거하기위해 토큰은 인자로 주입받는다.*

### 2. 주문/재고 Fetcher 클래스 구성
- [ ] `app/adapters/external/cafe24/client.py`:
  ```python
  import httpx
  from typing import List
  from app.schemas.integration_schema import CreateOrderDTO
  # (재고 DTO 인클루드 등)

  class Cafe24Adapter:
      def __init__(self, mall_id: str, access_token: str):
          self.mall_id = mall_id
          self.access_token = access_token
          self.base_url = f"https://{mall_id}.cafe24api.com/api/v2"
          
      async def get_orders(self, start_date: str, end_date: str) -> List[CreateOrderDTO]:
          """ 특정 시간의 주문 리스트 페이징 수집 """
          pass
          
      async def get_inventories(self, product_nos: List[str]) -> List[Any]:
          """ 특정 상품들의 남은 수량 수집 """
          pass
  ```

### 3. Rate Limit 폴링 및 재시도 방어기제 (Circuit Breaker)
- [ ] **가장 중요**: Response Header 의 `X-RateLimit-Requests` 혹은 429 에러 감지 로직 구비.
- [ ] 429 감지시 즉각 Fail 파기 하지 않고 `asyncio.sleep(N)` 이후 재시도(Retry)를 최대 3회 스로틀링 수행 (REQ-FUNC-015 준수 목적).

## 🧪 Acceptance Criteria (BDD/GWT)

**Scenario 1: Rate Limit 스로틀링 작동 검증**
- Given: 카페24 API 분당 콜 오버 헤드로 인한 429 응답 에러 상황
- When: `get_orders` 등 반복 배치 함수에서 HTTP 응답코드 429 인지
- Then: 로직이 크래쉬 나지 않고 `asyncio.sleep(초)` 기반 백오프 대기 후 재요청을 던져 성공적인 JSON을 받아오며 스로틀링 방어를 입증한다.

**Scenario 2: 응답 전문 파싱**
- Given: 카페24 복잡한 중첩 JSON 응답 구조물
- When: 내부 파서가 처리함
- Then: 우리 프로젝트 시스템 모델인 `CreateOrderDTO` 와 `OrderItemDTO` 규격 리스트로 직렬화 세탁되어 결과물이 순수하게 도출된다.

## ⚙️ Technical Constraints
- 카페24 엑세스 토큰은 2시간 만료 속성을 가지므로, 호출 전 `mall_id` 정보만 가지고 Redis 등 캐시에서 토큰을 꺼내보거나, 기간체크를 하는 프록시 로직을 어댑터 사용처(Service 영역)에 배치해 두어야 한다.

## 🏁 Definition of Done (DoD)
- [ ] `Cafe24Adapter` 클래스와 주문, 재고 Fetch 메소드 작성 ?
- [ ] 429 에러 감지 시의 `asyncio.sleep` Retrying 제어 흐름 구현?
- [ ] OAuth 토큰 헤더 갱신 헬퍼 모듈 분리?
