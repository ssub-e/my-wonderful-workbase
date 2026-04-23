---
type: concept
tags: [tech-stack, architecture, constraints]
created: 2026-04-22
updated: 2026-04-22
---
# C-TEC 기술 스택 및 개발 제약사항

MVP 단계에서 오버 엔지니어링을 막고, 향후 스케일업(Phase 2)으로의 매끄러운 이전을 위해 설정된 엄격한 단일 언어(Python) 중심의 기술 스택 원칙입니다.

## 1. 프론트엔드 및 백엔드 (Python 100%)
- **프론트엔드 (대시보드)**: `Streamlit` 사용. JavaScript 프레임워크 학습 및 세팅 시간을 배제하고 파이썬 백엔드 개발자 혼자서 UI를 전부 구축함. (향후 React+Recharts 전환 목표)
- **백엔드 프레임워크**: `FastAPI`. 비동기 기반 고성능 처리 및 Swagger/OpenAPI 자동화 지원.
- **PDF 렌더링**: `WeasyPrint` 또는 `FPDF2` 활용. (Puppeteer 등의 JS 런타임 도입을 유보함)

## 2. 데이터베이스 및 캐시
- **RDBMS**: `PostgreSQL` + `SQLModel` ORM. 테넌트 간 격리는 물리적 분리가 아닌 `tenant_id` 필드 기반의 어플리케이션 레벨 격리를 준수함.
- **인메모리 캐시**: `Redis` (Upstash 무료 티어 등). 외부 API Fallback 임시 저장 및 알림 큐 용도로 활용.
- **ETL 스케줄링**: `APScheduler`. FastAPI 인스턴스 내장 스케줄러로 기동하되, 휘발성 방지를 위해 PostgreSQL jobstore를 활용함. (Phase 2에서 Airflow로 분리)

## 3. 머신러닝 (AI 엔진)
- **코어 알고리즘**: `LightGBM` (1순위) 및 `Prophet`. Docker 컨테이너 사이즈를 가볍게 유지하고 빌드 시간을 10초 내로 통제하기 위해 무거운 딥러닝(TFT/N-BEATS)은 Phase 2로 연기함.
- **XAI 엔진**: `SHAP`. 변수별 기여도 수치를 도출하는 역할.
- **LLM 오케스트레이션**: `Google Gemini API`. SHAP 수치를 "임원진이 이해하기 쉬운 비즈니스 톤"으로 번역하는 자연어 렌더링 엔진으로 사용.

## 4. 인프라 운영
- **컨테이너**: `Docker` + `docker-compose`. OS 환경 간 차이를 원천 차단하기 위해 100% 도커 기반 로컬 개발.
- **배포 플랫폼**: `Railway` 또는 `Render`. 단일 서버에 Front/Back/DB/Scheduler를 몽땅 올려 MVP 인프라 유지 비용을 극단적으로 제어함. (향후 AWS ECS 전환)

## Related
- [기술 아키텍처 다이어그램 및 파이프라인](file:///e:/workspace/my-wonderful-workbase/wiki/concepts/system_arch/tech-architecture.md)
- [API 및 연동 스펙 명세서](file:///e:/workspace/my-wonderful-workbase/wiki/concepts/system_arch/api-specs.md)
- [원본 출처: SRS-V1.0 요약](file:///e:/workspace/my-wonderful-workbase/wiki/sources/specification/source-srs-v1.0.md)
