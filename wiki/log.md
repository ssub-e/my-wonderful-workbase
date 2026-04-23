# Timeline Log

위키에 수행된 모든 수집(Ingest), 생성, 점검(Lint) 기록입니다.

## [2026-04-22] 위키 인프라 초기화
- `CLAUDE.md` 스키마 및 컨벤션 문서 생성
- `index.md` 카탈로그 문서 생성
- `log.md` 타임라인 기록 문서 생성

## [2026-04-22] Ingest | 07_VPS_Final_for_PRD
- [source-vps.md](file:///e:/workspace/my-wonderful-workbase/wiki/sources/source-vps.md) 요약본 생성
- [personas.md](file:///e:/workspace/my-wonderful-workbase/wiki/entities/personas.md) 엔티티 추출
- [value-proposition.md](file:///e:/workspace/my-wonderful-workbase/wiki/concepts/value-proposition.md) 컨셉 추출

## [2026-04-22] Ingest & Node Splitting | 비즈니스 분석 01~10번 대량 수집
- **Batch 1 (01, 02, 06번)**: 시장 구조(5 Forces), 경쟁 포지셔닝(Competitor), TAM/SAM/SOM 수치 분석 노드 분리
- **Batch 2 (03, 04, 05번)**: 가치사슬 벤치마킹, KSF(핵심 성공 요인), B2B 수요예측 최종 문제 정의서 노드 분리
- **Batch 3 (07, 08, 09, 10번)**: 극단적 분기형 CJM, 기회점수(AOS/DOS) 매트릭스, JTBD 심층 인터뷰 내용 추출 및 엔티티 연결
## [2026-04-22] Ingest & Node Splitting | SRS-V1.0.md
- **Tech Stack & Constraints**: SRS 기반 파이썬 단일 MVP 기술 스택(C-TEC) 정책 추출
- **API Spec**: Internal 및 External API 인터페이스 엔티티 분리
- `index.md` 구조에 C-TEC 및 API 스펙 노드 배치, Phase 2 (핵심 수집) 완료
- **Aggressive Structure**: PRD를 해체하여 다차원 폴더 노드로 생성
- [source-prd-v1.1.md](file:///e:/workspace/my-wonderful-workbase/wiki/sources/specification/source-prd-v1.1.md) 요약 및 연결점(Bridge) 생성
- [erd-core-entities.md](file:///e:/workspace/my-wonderful-workbase/wiki/entities/database/erd-core-entities.md) DB 엔티티 추출
- [tech-architecture.md](file:///e:/workspace/my-wonderful-workbase/wiki/concepts/system_arch/tech-architecture.md) 기술 파이프라인 추출
- [adr-nfr.md](file:///e:/workspace/my-wonderful-workbase/wiki/concepts/system_arch/adr-nfr.md) 아키텍처 제약 및 의사결정 추출
- [kpi-and-benchmarks.md](file:///e:/workspace/my-wonderful-workbase/wiki/concepts/product_metrics/kpi-and-benchmarks.md) 핵심 지표 및 실험 설계 추출

## [2026-04-22] Task Mapping | GitHub Issues
- **Tasks Migration**: `e:/workspace/SRS-from-PRD/Tasks/issues/` 폴더에 있던 146개의 마크다운 태스크 문서를 `wiki/tasks/` 구조로 복사/분류 (DB, API, VIEW 등 prefix 기반)
- **Frontmatter Addition**: Obsidian 인식을 위한 `type: task` 및 `tags` 메타데이터 부여 완료
- **Task Indexing**: 전체 146개 태스크를 그룹핑한 `index-tasks.md` 생성 및 `wiki/index.md`에 연동. Phase 3 완료.
