---
name: Feature Task
about: SRS 기반의 구체적인 개발 태스크 명세
title: "[Infra] ENV-002: 프로젝트 디렉토리 구조 설계 — 헥사고날 아키텍처"
labels: 'infra, backend, architecture, priority:critical, phase:0'
assignees: ''
type: task
tags: [task, infrastructure]
created: 2026-04-22
---
## 🎯 Summary
- 기능명: [ENV-002] 프로젝트 디렉토리 구조 설계 (헥사고날 아키텍처 모듈 경계)
- 목적: REQ-NF-029(헥사고날 아키텍처) 및 REQ-NF-026(모듈 독립성)에 따라, 외부 시스템(DB, API) 변경이 핵심 비즈니스 로직에 전파되지 않도록 격리된 디렉토리 구조를 확립한다. AI 에이전트와 개발자가 일관된 룰에 따라 코드를 배치할 수 있는 물리적 뼈대를 구성한다.

## 🔗 References (Spec & Context)
> 💡 AI Agent & Dev Note: 작업 시작 전 아래 문서를 반드시 먼저 Read/Evaluate 할 것.
- SRS 아키텍처: [`SRS-V1.0.md#§3.6`](file:///e:/workspace/SRS-from-PRD/SRS-V1.0.md) — Component Architecture Diagram
- SRS 모듈 독립성: [`SRS-V1.0.md#§4.2.6`](file:///e:/workspace/SRS-from-PRD/SRS-V1.0.md) — REQ-NF-029 (헥사고날), REQ-NF-026 (모듈 분리)
- **제약사항:** C-TEC-002 (FastAPI), C-TEC-013 (Docker 통합 환경)

## ✅ Task Breakdown (실행 계획)

### 1. 헥사고날 루트 구성
- [ ] 프로젝트 최상위에 `app/`, `tests/`, `scripts/`, `docs/` 폴더를 생성하고 `__init__.py` 구성.

### 2. 세부 디렉토리 트리 확립 (의존성 규칙 기반)
- [ ] 아래 명시된 구조에 맞춰 모든 폴더 생성:
```text
app/
├── main.py                   # 애플리케이션 진입점 (FastAPI 인스턴스)
├── core/                     # 공통 인프라 (DB 연결, 설정, 보안, 스케줄러)
│   ├── config.py             # Pydantic Settings
│   ├── database.py           # DB 엔진 및 세션 팩토리
│   ├── security.py           # JWT 및 패스워드 유틸
│   └── scheduler.py          # APScheduler 인스턴스
├── domain/                   # 순수 비즈니스 로직 (Port, Entity, Service)
│   ├── _common/              # 공통 Base Model 등
│   ├── tenant/               # 테넌트 도메인 (DB-001)
│   ├── auth/                 # 인증/권한 도메인 (AUTH-001)
│   ├── integration/          # 쇼핑몰 연동/주문/재고 도메인 (F2)
│   ├── forecast/             # 수요 예측 및 XAI 도메인 (F1)
│   └── workforce/            # 인원 역산 도메인 (F3)
├── adapters/                 # 외부 시스템 연동 (Adapter)
│   ├── persistence/          # DB 리포지토리 구현체
│   ├── external/             # 외부 API 통신 (기상청, 카페24, 카카오 등)
│   └── llm/                  # Gemini LLM 어댑터
├── api/                      # Inbound Adapter (라우터 Controller)
│   ├── v1/
│   │   ├── auth.py
│   │   ├── forecast.py
│   │   ├── integration.py
│   │   └── dashboard.py
│   └── router.py             # 상위 라우터 통합
├── schemas/                  # Pydantic DTO (Request / Response)
│   ├── tenant_schema.py
│   ├── forecast_schema.py
│   └── integration_schema.py
└── middleware/               # HTTP 필터 (테넌트 식별, 로깅, 예외 처리)
    └── auth_middleware.py
```

### 3. Architecture Guidelines 문서화
- [ ] `docs/architecture_guidelines.md` 작성:
  - 의존성 흐름 강제: `adapters`, `api` ➔ `domain` (도메인은 절대 바깥 계층을 import 하지 않음)
  - 엔터티와 DTO 분리 원칙: `domain/` 안에는 SQLModel 엔터티 로직만, API의 입출력은 `schemas/` 의 Pydantic을 거쳐 변환.

## 🧪 Acceptance Criteria (BDD/GWT)

**Scenario 1: 디렉토리 접근성 검증**
- Given: Python (`pytest`) 초기 환경
- When: 애플리케이션 기동 시 `from app.domain.integration.models import Shop` 등 구조적 임포트를 시도함
- Then: 모든 모듈 경로가 `ModuleNotFoundError` 없이 정상 파싱된다.

**Scenario 2: Linter 가이드라인 (의존성 격리)**
- Given: `app/domain/tenant/models.py` 파일 내부
- When: 개발자가 `from app.api.v1.auth import ...` 구문을 오작성함
- Then: 아키텍처 규칙 위반으로 코드 리뷰 시 반려(Reject)될 수 있도록 구조와 규칙 문서가 명확히 가이드한다.

## ⚙️ Technical Constraints
- **절대 경로 사용**: Python 패키지 내 상대 경로 임포트(`from ..domain import`)를 지양하고, 루트 기반 절대 경로(`from app.domain import`)를 표준으로 강제.

## 🏁 Definition of Done (DoD)
- [ ] 위의 전체 폴더 트리가 빈 폴더 형태로 레포지토리에 커밋(`__init__.py` 포함) 되었는가?
- [ ] `docs/architecture_guidelines.md` 문서가 작성되어 의존성 제약을 명시하는가?

## 🚧 Dependencies
- **Depends on**: ENV-001 (동일한 저장소 뿌리 공유)
- **Blocks**: ENV-006 (FastAPI 앱이 이 디렉토리 구조 위에 파일들로 세팅됨)
