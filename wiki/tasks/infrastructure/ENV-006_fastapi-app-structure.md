---
name: Feature Task
about: SRS 기반의 구체적인 개발 태스크 명세
title: "[Infra] ENV-006: FastAPI 애플리케이션 기본 구조 세팅"
labels: 'infra, backend, priority:critical, phase:0'
assignees: ''
type: task
tags: [task, infrastructure]
created: 2026-04-22
---
## 🎯 Summary
- 기능명: [ENV-006] FastAPI 애플리케이션 기본 구조 세팅
- 목적: 헥사고날 아키텍처(REQ-NF-029) 원칙에 따라 FastAPI 백엔드 애플리케이션의 뼈대를 구축한다. 라우터, 미들웨어, 에러 핸들러, OpenAPI 자동 문서를 포함한 프로젝트 기초 구조를 확립하여, 이후 모든 Feature Task가 이 뼈대 위에서 일관되게 개발될 수 있도록 한다.

## 🔗 References (Spec & Context)
> 💡 AI Agent & Dev Note: 작업 시작 전 아래 문서를 반드시 먼저 Read/Evaluate 할 것.
- SRS 문서: [`SRS-V1.0.md#§1.5.3`](file:///e:/workspace/SRS-from-PRD/SRS-V1.0.md) — C-TEC-002 FastAPI
- SRS 아키텍처: [`SRS-V1.0.md#§3.6`](file:///e:/workspace/SRS-from-PRD/SRS-V1.0.md) — Component Architecture (API Layer)
- SRS API 목록: [`SRS-V1.0.md#§3.3`](file:///e:/workspace/SRS-from-PRD/SRS-V1.0.md) — API-INT-01~10 개요
- SRS API 상세: [`SRS-V1.0.md#§6.1`](file:///e:/workspace/SRS-from-PRD/SRS-V1.0.md) — 전체 Endpoint List
- SRS 모듈 독립성: [`SRS-V1.0.md#§4.2.6`](file:///e:/workspace/SRS-from-PRD/SRS-V1.0.md) — REQ-NF-026 모듈 독립성, REQ-NF-029 헥사고날 아키텍처
- **관련 C-TEC 제약:**
  - C-TEC-002: FastAPI — 비동기 고성능 REST API, 자동 OpenAPI 문서 생성
  - C-TEC-003: PostgreSQL + SQLModel
  - C-TEC-009: Google Gemini API — 어댑터 패턴 적용

## ✅ Task Breakdown (실행 계획)

### 프로젝트 디렉토리 구조 (헥사고날 아키텍처)
- [ ] 최상위 디렉토리 구조 확립:
```
app/
├── main.py                    # FastAPI 앱 팩토리
├── core/
│   ├── config.py              # Pydantic Settings (환경 변수)
│   ├── database.py            # SQLModel 세션 + 엔진
│   ├── security.py            # JWT, OAuth, 비밀번호 해싱
│   └── dependencies.py        # 공통 의존성 주입
├── domain/                    # 도메인 모델 (Port — 순수 비즈니스)
│   ├── forecast/
│   ├── integration/
│   ├── workforce/
│   └── report/
├── adapters/                  # 어댑터 (외부 시스템 연동)
│   ├── external/              # 기상청, 카페24, 네이버, 카카오 등
│   └── persistence/           # DB 리포지토리
├── api/                       # API 라우터 (Inbound Adapter)
│   └── v1/
│       ├── forecast.py
│       ├── report.py
│       ├── integration.py
│       ├── workforce.py
│       ├── notification.py
│       ├── dashboard.py
│       └── xai.py
├── middleware/                 # 미들웨어
│   ├── tenant.py              # 멀티테넌트 tenant_id 필터링
│   ├── auth.py                # JWT 인증
│   └── logging.py             # 요청/응답 로깅
├── schemas/                   # Pydantic DTO (Request/Response)
└── tests/                     # 테스트
    ├── unit/
    └── integration/
```

### FastAPI 앱 기초 구현
- [ ] `main.py` — FastAPI 앱 팩토리 패턴 (`create_app()`) 구현
  - [ ] CORS 미들웨어 설정
  - [ ] API 라우터 v1 그룹 등록 (`/api/v1/` prefix)
  - [ ] OpenAPI 문서 메타데이터 (title, version, description)
  - [ ] lifespan 이벤트 핸들러 (startup: DB 연결, Redis 연결 / shutdown: 정리)
- [ ] `core/config.py` — Pydantic BaseSettings 기반 설정 모듈
  - [ ] DATABASE_URL, REDIS_URL, SECRET_KEY, GEMINI_API_KEY 등 필수 환경 변수 정의
  - [ ] `.env` 파일 자동 로드
  - [ ] 필수 값 누락 시 ValidationError 발생
- [ ] `core/database.py` — SQLModel 세션 및 엔진 설정
  - [ ] `get_session()` 의존성 주입 함수 (async generator)
  - [ ] 커넥션 풀 설정 (pool_size, max_overflow)
- [ ] `middleware/logging.py` — 요청/응답 구조화 로깅 미들웨어
  - [ ] request_id 자동 생성 (UUID)
  - [ ] 요청 메서드, URL, 응답 코드, 소요시간 로깅
- [ ] 글로벌 에러 핸들러 — HTTPException, ValidationError, 500 Internal Server Error 통합 처리
  - [ ] 표준화된 에러 응답 포맷: `{"error": {"code": "...", "message": "...", "details": [...]}}`
- [ ] `/health` 헬스체크 엔드포인트 — DB, Redis 연결 상태 확인
- [ ] `/api/v1/` 하위 빈 라우터 스텁 7개 등록 (forecast, report, integration, workforce, notification, dashboard, xai)

## 🧪 Acceptance Criteria (BDD/GWT)

**Scenario 1: 앱 기동 및 OpenAPI 문서 접근**
- Given: ENV-001의 Docker 환경이 정상 기동된 상태
- When: `http://localhost:8000/docs` 에 접속함
- Then: Swagger UI가 렌더링되고, 7개 API 그룹(forecast, report, integration, workforce, notification, dashboard, xai)의 빈 엔드포인트 스텁이 표시된다.

**Scenario 2: 헬스체크 성공**
- Given: PostgreSQL, Redis 컨테이너가 모두 healthy 상태일 때
- When: `GET /health` 요청
- Then: `200 OK` + `{"status": "ok", "db": "connected", "redis": "connected"}` 응답 반환.

**Scenario 3: 헬스체크 DB 실패**
- Given: PostgreSQL 컨테이너가 중지된 상태일 때
- When: `GET /health` 요청
- Then: `503 Service Unavailable` + `{"status": "degraded", "db": "disconnected", "redis": "connected"}` 응답 반환.

**Scenario 4: 글로벌 에러 핸들러 동작**
- Given: 존재하지 않는 엔드포인트에 요청 시
- When: `GET /api/v1/nonexistent` 요청
- Then: `404 Not Found` + 표준화된 에러 JSON 포맷 `{"error": {"code": "NOT_FOUND", "message": "..."}}` 응답 반환.

**Scenario 5: 환경 변수 누락 시 기동 실패**
- Given: `DATABASE_URL` 환경 변수가 설정되지 않은 상태
- When: FastAPI 앱 기동 시도
- Then: `ValidationError`와 함께 누락된 변수명을 포함한 명확한 에러 메시지가 출력되며 앱이 기동되지 않는다.

**Scenario 6: 요청 로깅 미들웨어 작동**
- Given: 로깅 미들웨어가 활성화된 상태
- When: 임의의 API 요청을 보냄
- Then: 로그에 `request_id`, HTTP 메서드, URL 경로, 응답 코드, 소요시간(ms)이 구조화되어 출력된다.

## ⚙️ Technical & Non-Functional Constraints
- **C-TEC-002 준수**: FastAPI 사용, 자동 OpenAPI 문서 생성 (Swagger/ReDoc)
- **REQ-NF-029 준수**: 헥사고날 아키텍처 — 외부 라이브러리 변경이 도메인 코어에 전파되지 않는 구조
- **REQ-NF-026 준수**: 모듈 독립성 — 각 도메인(forecast, integration, workforce, report)은 독립 모듈로 분리, 개별 배포 가능한 모듈 경계
- **REQ-NF-027 준수**: 외부 API 어댑터 패턴 — `adapters/external/` 하위에 각 외부 시스템별 Port + Adapter 분리
- **비동기 우선**: async/await 기반 비동기 처리 기본 적용
- **Python 3.11+**: 타입 힌팅 적극 활용, Pydantic v2 사용

## 🏁 Definition of Done (DoD)
- [ ] 모든 Acceptance Criteria(6개 시나리오)를 충족하는가?
- [ ] 헥사고날 아키텍처 디렉토리 구조가 확립되었는가? (domain / adapters / api 분리)
- [ ] `GET /docs`에서 Swagger UI가 정상 렌더링되는가?
- [ ] `GET /health`에서 DB + Redis 연결 상태가 확인되는가?
- [ ] 글로벌 에러 핸들러가 표준화된 JSON 포맷으로 응답하는가?
- [ ] `core/config.py`의 필수 환경 변수 누락 시 명확한 에러를 반환하는가?
- [ ] 로깅 미들웨어가 request_id와 함께 구조화된 로그를 출력하는가?
- [ ] 7개 API 라우터 스텁이 `/api/v1/` 하위에 등록되어 있는가?
- [ ] 단위 테스트(pytest)가 최소 1개 이상 작성되고 통과하는가?

## 🚧 Dependencies & Blockers
- **Depends on**:
  - ENV-001 (Docker 개발 환경 — 컨테이너 기동 필수)
  - ENV-002 (프로젝트 디렉토리 구조 설계 — 헥사고날 아키텍처 원칙 확립)
- **Blocks**:
  - DB-001~011 (전체 데이터 모델 — database.py 세션 의존)
  - API-001~010 (전체 API 계약 — 라우터 스텁 위에 구현)
  - AUTH-001 (인증 모듈 — security.py 위에 구현)
  - AUDIT-001 (감사 로그 — 미들웨어 위에 구현)
  - 모든 Feature Task (F1/F2/F3) — 이 뼈대 위에서 개발됨
