---
name: Feature Task
about: SRS 기반의 구체적인 개발 태스크 명세
title: "[Infra] ENV-001: Docker + docker-compose 개발 환경 표준화"
labels: 'infra, devops, priority:critical, phase:0'
assignees: ''
type: task
tags: [task, infrastructure]
created: 2026-04-22
---
## 🎯 Summary
- 기능명: [ENV-001] Docker + docker-compose 개발 환경 표준화
- 목적: 모든 개발자와 AI 에이전트가 OS 차이 없이 동일한 환경에서 개발할 수 있도록, Docker 기반의 컨테이너화된 개발 환경을 Sprint 0에서 확립한다. 로컬 ≈ 배포 환경 일치를 보장하여 "내 PC에서는 되는데" 문제를 원천 차단한다.

## 🔗 References (Spec & Context)
> 💡 AI Agent & Dev Note: 작업 시작 전 아래 문서를 반드시 먼저 Read/Evaluate 할 것.
- SRS 문서: [`SRS-V1.0.md#§1.5.3`](raw/assets/SRS-V1.0.md) — C-TEC-013 개발 환경 표준
- SRS 기술 스택: [`SRS-V1.0.md#§1.5.3`](raw/assets/SRS-V1.0.md) — C-TEC-001~014 전체 스택
- SRS 아키텍처: [`SRS-V1.0.md#§3.6`](raw/assets/SRS-V1.0.md) — Component Architecture Diagram
- **관련 C-TEC 제약:**
  - C-TEC-002: FastAPI (Backend API)
  - C-TEC-003: PostgreSQL + SQLModel (DB)
  - C-TEC-004: Redis / Upstash (캐시)
  - C-TEC-008: APScheduler (ETL 스케줄링)
  - C-TEC-013: Docker + docker-compose (개발 환경 표준)

## ✅ Task Breakdown (실행 계획)
- [ ] `Dockerfile` 작성 — Python 3.11+ 베이스 이미지, FastAPI + 의존성 설치, 포트 설정
- [ ] `docker-compose.yml` 작성 — 아래 서비스 정의:
  - [ ] `app`: FastAPI 애플리케이션 (포트 8000)
  - [ ] `db`: PostgreSQL 16 (포트 5432, 볼륨 마운트)
  - [ ] `redis`: Redis 7 (포트 6379)
- [ ] `.env.example` 템플릿 작성 — DB 접속 정보, Redis URL, API 키 플레이스홀더
- [ ] `Makefile` 또는 스크립트 작성 — `make up`, `make down`, `make migrate`, `make test` 등 개발 편의 명령
- [ ] 볼륨 마운트 설정 — 소스 코드 핫 리로드 (uvicorn --reload)
- [ ] 헬스체크 엔드포인트 (`/health`) — DB 연결 + Redis 연결 확인
- [ ] `README.md` — 초기 설정 가이드 (git clone → docker compose up → 접속 확인)

## 🧪 Acceptance Criteria (BDD/GWT)

**Scenario 1: 최초 환경 구축 (Cold Start)**
- Given: 프로젝트를 처음 clone한 개발자 (또는 AI 에이전트)
- When: `docker compose up -d` 명령을 실행함
- Then: FastAPI 앱(8000), PostgreSQL(5432), Redis(6379) 세 컨테이너가 모두 healthy 상태로 기동되고, `http://localhost:8000/health` 접속 시 `{"status": "ok", "db": "connected", "redis": "connected"}` 응답을 반환한다.

**Scenario 2: 코드 변경 핫 리로드**
- Given: docker compose up 상태에서 앱 컨테이너가 실행 중일 때
- When: 소스 코드 파일(예: `main.py`)을 수정하고 저장함
- Then: uvicorn이 자동으로 서버를 재시작하고, 변경 사항이 즉시 반영된다 (컨테이너 재빌드 불필요).

**Scenario 3: 클린 종료 및 데이터 보존**
- Given: 개발 중 DB에 테스트 데이터가 저장된 상태
- When: `docker compose down` 후 다시 `docker compose up -d` 실행
- Then: PostgreSQL 볼륨 마운트로 인해 기존 DB 데이터가 보존된다.

**Scenario 4: 환경 변수 미설정 시 실패**
- Given: `.env` 파일이 없거나 필수 환경 변수가 누락된 상태
- When: `docker compose up` 실행
- Then: 명확한 에러 메시지와 함께 앱이 기동에 실패하고, 어떤 환경 변수가 누락 되었는지 안내한다.

## ⚙️ Technical & Non-Functional Constraints
- **C-TEC-013 준수**: 모든 개발은 컨테이너 내에서 진행하여 OS 차이 원천 차단
- **Python 단일 스택**: Python 3.11+ 베이스 이미지 사용 (C-TEC 전체 원칙)
- **이미지 경량화**: LightGBM Docker 이미지 빌드 < 10초 목표 (C-TEC-005)
- **배포 호환**: Railway / Render 배포와 동일한 Docker 이미지 사용 (C-TEC-010)
- **보안**: `.env` 파일은 `.gitignore`에 포함, 시크릿 하드코딩 절대 금지

## 🏁 Definition of Done (DoD)
- [ ] 모든 Acceptance Criteria(4개 시나리오)를 충족하는가?
- [ ] `docker compose up -d` 한 번으로 전체 개발 환경이 기동되는가?
- [ ] PostgreSQL, Redis 컨테이너가 healthy 상태인가?
- [ ] `/health` 엔드포인트가 DB + Redis 연결 상태를 정상 반환하는가?
- [ ] `.env.example`에 모든 필수 환경 변수가 문서화되어 있는가?
- [ ] `README.md`에 초기 설정 步순이 작성되어 있는가?
- [ ] `.gitignore`에 `.env`, `__pycache__`, `.venv` 등이 포함되어 있는가?

## 🚧 Dependencies & Blockers
- **Depends on**: None (최초 Task — 프로젝트 시작점)
- **Blocks**:
  - ENV-002 (프로젝트 디렉토리 구조 설계)
  - ENV-003 (CI/CD 파이프라인 구성)
  - ENV-004 (Python 의존성 관리)
  - ENV-005 (환경 변수 관리 체계)
  - ENV-006 (FastAPI 기본 구조)
  - INFRA-001 (PostgreSQL 프로덕션 DB 설정)
  - INFRA-002 (Redis 캐시 레이어 설정)
  - DB-001~011 (전체 데이터 모델)
