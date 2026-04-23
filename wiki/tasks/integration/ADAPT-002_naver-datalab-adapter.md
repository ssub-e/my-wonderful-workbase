---
name: Feature Task
about: SRS 기반의 구체적인 개발 태스크 명세
title: "[Adapter] ADAPT-002: 네이버 DataLab 트렌드 API 어댑터"
labels: 'adapter, external-api, phase:1, priority:medium'
assignees: ''
type: task
tags: [task, integration]
created: 2026-04-22
---
## 🎯 Summary
- 기능명: [ADAPT-002] 네이버 DataLab 검색어 트렌드 분석 API 연동
- 목적: REQ-FUNC-002 규칙에 따라 타겟 품목 군의 검색량 추이를 매일 1회 수집한다. 특정 기간 내의 '검색 지수(0~100)'를 반환하는 구조를 파싱하여 모델에 주입하기 위함이다.

## 🔗 References (Spec & Context)
- SRS 수집 요구: [`SRS-V1.0.md#§4.1.1`](file:///e:/workspace/SRS-from-PRD/SRS-V1.0.md) — REQ-FUNC-002
- 네이버 Developers API 가이드라인 연동

## ✅ Task Breakdown (실행 계획)

### 1. HTTPX 기반 통신 모듈 구축
- [ ] `app/adapters/external/naver_datalab.py`:
  ```python
  import httpx
  from typing import List, Dict
  from app.schemas.forecast_schema import TrendDataCreate

  class NaverDataLabAdapter:
      def __init__(self, client_id: str, client_secret: str):
          self.headers = {
              "X-Naver-Client-Id": client_id,
              "X-Naver-Client-Secret": client_secret,
              "Content-Type": "application/json"
          }
          self.url = "https://openapi.naver.com/v1/datalab/search"
          
      async def fetch_trend_index(self, keyword: str, start_date: str, end_date: str) -> List[TrendDataCreate]:
          body = {
              "startDate": start_date,
              "endDate": end_date,
              "timeUnit": "date",
              "keywordGroups": [{"groupName": keyword, "keywords": [keyword]}]
          }
          # async with httpx.AsyncClient() ...
  ```

### 2. 응답 맵핑 로직 마련
- [ ] 네이버의 응답 `data -> [ {period: '', ratio: float} ]` 구조를 순회하며 `TrendDataCreate` 의 리스트 형태로 전부 변환하여 반환.

### 3. 일일 할당량 제어 방어
- [ ] 리턴 코드가 `429 Too Many Requests` (할당량 초과) 등 에러일 경우 시스템 다운 방어를 위해 명확한 `QuotaExceededError` 핸들링 처리.

## 🧪 Acceptance Criteria (BDD/GWT)

**Scenario 1: 정상 검색량 지수 변환**
- Given: `"마스크"` 키워드에 대해 `2026-04-01` ~ `2026-04-03` 범례 조회를 지시함
- When: 어댑터 호출
- Then: 네이버에서 응답된 각 일별 ratio 값을 딴 길이 3짜리 `[TrendDataCreate, TrendDataCreate...]` 컨테이너가 깔끔하게 반환된다.

**Scenario 2: API 접근 키 불량/만료/과조회**
- Given: 잘못 지정된 `CLIENT_SECRET` 로 셋업된 객체 상태
- When: 호출 시도
- Then: 401 Unauthorized 에러 코드를 내부 커스텀 예외로 묶어(`ExternalAPIAuthError`) 상위 ETL 에러 리포터 층에 즉각 토스한다.

## ⚙️ Technical Constraints
- 어댑터 내부에서 HTTP Client Connection Pool (`httpx.AsyncClient()`) 을 반복 생성 소멸 하지 않고, 어플리케이션 Lifespan에서 싱글턴으로 물려주거나 적절한 커넥션 재사용 전략(Session 유지)을 통해 소켓 누수와 속도 손실을 방지할 것.

## 🏁 Definition of Done (DoD)
- [ ] `NaverDataLabAdapter` 기능 작성 (Post 요청 바디 포맷 포함) 확인?
- [ ] 401, 429 등 주요 API Status 코드를 방어하는 예외 핸들링 구조 구축?
- [ ] `settings` 모듈을 통한 ID/PWD 의존성 주입 여부 확인?
