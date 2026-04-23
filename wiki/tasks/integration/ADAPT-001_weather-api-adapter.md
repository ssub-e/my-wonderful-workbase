---
name: Feature Task
about: SRS 기반의 구체적인 개발 태스크 명세
title: "[Adapter] ADAPT-001: 기상청 단기예보 OpenAPI 어댑터"
labels: 'adapter, external-api, phase:1, priority:high'
assignees: ''
type: task
tags: [task, integration]
created: 2026-04-22
---
## 🎯 Summary
- 기능명: [ADAPT-001] 기상청 단기예보 API 통신 어댑터 구성
- 목적: REQ-FUNC-001에 따라 매일 1회 익일의 지역별 기온/강수량 데이터를 추출한다. 헥사고날 아키텍처에 입각해 기상청의 지저분한 XML/JSON 응답 포맷을 어플리케이션 내부 도메인 양식(`WeatherDataCreate`)으로 깔끔히 변환하는 통신 객체를 생성한다.

## 🔗 References (Spec & Context)
- SRS 수집 요구: [`SRS-V1.0.md#§4.1.1`](raw/assets/SRS-V1.0.md) — REQ-FUNC-001
- 신뢰성 폴백: [`SRS-V1.0.md#§4.2.2`](raw/assets/SRS-V1.0.md) — REQ-NF-010 (기상청 응답 지연/에러 시 캐시 폴백 체계)
- 헥사고날 룰: `docs/architecture_guidelines.md`

## ✅ Task Breakdown (실행 계획)

### 1. 통신 어댑터 클래스 정의
- [ ] `app/adapters/external/weather_api.py` 파일 생성:
  ```python
  import httpx
  from app.core.config import settings
  from app.schemas.forecast_schema import WeatherDataCreate

  class WeatherAPIAdapter:
      def __init__(self, api_key: str):
          self.api_key = api_key
          self.base_url = "http://apis.data.go.kr/1360000/VilageFcstInfoService_2.0/getVilageFcst"
          
      async def fetch_forecast(self, nx: str, ny: str, target_date: str) -> WeatherDataCreate:
          # httpx.AsyncClient 를 이용한 네트워크 I/O 실행
          # 응답받은 기상청 JSON 규격 파싱 및 정제
          # app.schemas 표준 Pydantic 객체로 변환하여 리턴
          pass
  ```

### 2. 기상청 고유 스펙 컨버터 작성 (좌표계 변환)
- [ ] 서울/부산 등 시군구 명칭을 기상청 고유 그리드 XY 좌표(nx, ny)로 맵핑해주는 유틸리티 내장 (`region_to_grid(region: str) -> Tuple[int, int]`).

### 3. 오류 대응 및 재시도 (Circuit Breaker 기반)
- [ ] 공공 API 특성상 Timeout 빈번함 -> HTTPX에 `timeout=10.0` 필수 부여.
- [ ] 접속 실패나 전문 오류 발생 시 `WeatherAPIError` 커스텀 커스텀 예외 raise. (이를 트리거로 로직 윗단에서 Redis Cache 폴백 작동하게 됨).

## 🧪 Acceptance Criteria (BDD/GWT)

**Scenario 1: 정상 수집 및 도메인 모델 변환**
- Given: 기상청 API 활성화 키 확보 및 정상 응답 상황
- When: 어댑터에 `nx=60, ny=127` (서울)과 `base_date=20260501` 파라미터 전달 후 호출
- Then: 기상청 JSON이 파싱되어 내부 의존성이 없는 순수 DTO 형태인 `WeatherDataCreate(region="seoul", temperature=15.5, ...)` 로 리턴된다.

**Scenario 2: 공공 데이터 패킷 드랍 예외 처리 (Timeout Timeout)**
- Given: 기상청 서버 점검 또는 트래픽 단절 상황
- When: HTTP Client (httpx) 가 리퀘스트를 날림
- Then: 10 초 후 무한정 행이 걸리지 않도록 `ReadTimeout` 이 발생하며 이를 Catch한 어댑터가 명시적 예외 처리(`WeatherAPIError`)를 상위 서킷으로 던진다.

## ⚙️ Technical Constraints
- 라이브러리: 비동기 논블로킹 I/O를 위해 `requests` 모듈 사용 금지. 반드시 `httpx.AsyncClient` 컨텍스트를 활용한다.
- 헥사고날 규칙: 이 어댑터 안에서 데이터베이스 Repository 객체를 직접 임포트해서는 절대 안 된다. 오직 데이터를 Fetch 해서 변환해 리턴하는 포트 역할만 맡는다.

## 🏁 Definition of Done (DoD)
- [ ] `httpx` 비동기 기반의 기상 조회 어댑터 클래스 작성이 완료되었는가?
- [ ] API Key가 하드코딩 되지 않고 `settings.WEATHER_API_KEY` 환경변수로 주입되는가?
- [ ] 그리드 XY 맵퍼 로직 포함 및 Exception 처리 로직 구축?
