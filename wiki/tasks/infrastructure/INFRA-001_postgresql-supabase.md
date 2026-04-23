---
name: Feature Task
about: SRS 기반의 구체적인 개발 태스크 명세
title: "[Infra] INFRA-001: PostgreSQL + SQLModel 프로덕션 DB 설정 및 커넥션 풀링"
labels: 'infra, database, priority:critical, phase:1'
assignees: ''
type: task
tags: [task, infrastructure]
created: 2026-04-22
---
## 🎯 Summary
- 기능명: [INFRA-001] PostgreSQL + SQLModel 비동기 엔진 셋업 및 풀링 구성
- 목적: C-TEC-003(PostgreSQL)의 필수 기반으로서 동시다발적인 FastAPI 요청과 백그라운드 스케줄러(ETL) 접근을 I/O 블로킹 없이 처리하기 위한 비동기 `asyncpg` 엔진을 초기화한다. 커넥션 풀을 설정하여 RDBMS 장애에 유연하게 대처하며 RPO/RTO 백업 정책 기반을 설계한다.

## 🔗 References (Spec & Context)
- SRS DB 제약: [`SRS-V1.0.md#§1.5.3`](file:///e:/workspace/SRS-from-PRD/SRS-V1.0.md) — C-TEC-003 (PG + SQLModel, asyncpg 드라이버 권장)
- SRS 신뢰성 요구사항: [`SRS-V1.0.md#§4.2.2`](file:///e:/workspace/SRS-from-PRD/SRS-V1.0.md) — REQ-NF-013 (RPO 1시간 이내 백업), REQ-NF-014 (RTO 복구 절차)

## ✅ Task Breakdown (실행 계획)

### 1. 비동기 엔진 및 커넥션 풀 설정
- [ ] `app/core/database.py` 구현:
  ```python
  from sqlalchemy.ext.asyncio import create_async_engine, async_sessionmaker, AsyncSession
  from app.core.config import settings

  # PostgreSQL asyncpg 드라이버 필수
  engine = create_async_engine(
      settings.DATABASE_URL.replace("postgresql://", "postgresql+asyncpg://"),
      echo=settings.ECHO_SQL,
      pool_size=20,          # 동시 접속 최소 방어
      max_overflow=10,       # 트래픽 스파이크 시 추가 허용
      pool_pre_ping=True,    # 끊어진 세션 재연결 (유실 방어)
      pool_recycle=3600      # 1시간 주기로 커넥션 갱신
  )

  AsyncSessionLocal = async_sessionmaker(
      bind=engine,
      class_=AsyncSession,
      autocommit=False,
      autoflush=False,
      expire_on_commit=False
  )
  ```

### 2. 세션 제너레이터 의존성 주입
- [ ] FastAPI 의존성 주입용 제너레이터 작성:
  ```python
  async def get_session() -> AsyncGenerator[AsyncSession, None]:
      async with AsyncSessionLocal() as session:
          try:
              yield session
          except Exception:
              await session.rollback()
              raise
          finally:
              await session.close()
  ```

### 3. 클라우드 RPO/RTO 백업 스크립트화 (문서)
- [ ] 운영 지침서 `docs/database_operations.md` 작성. Supabase의 PITR(Point in Time Recovery) 설정 가이드 매뉴얼화 (목표 대상: SaaS 관리자용 RPO 1시간 복구 지침).

## 🧪 Acceptance Criteria (BDD/GWT)

**Scenario 1: 비동기 데이터베이스 연결 정상**
- Given: 올바른 환경변수 `DATABASE_URL` (로컬 Docker PG)
- When: FastAPI 앱 기동 및 `get_session` 호출
- Then: 비동기 커넥션 객체가 획득되고 간단한 `SELECT 1` 통과.

**Scenario 2: 예외 발생 시 안전한 세션 롤백**
- Given: 라우터 내에서 트랜잭션을 시작하고 DB 쓰기를 수행
- When: 코드 로직 내 에러 유발(Raise Exception)
- Then: `get_session` 제너레이터의 try-except가 작동하여 DB 상태가 롤백(rollback)되며, 세션은 유휴 자원 없이 리턴(close)된다.

**Scenario 3: 장기 유휴 끊김 방어(pre-ping)**
- Given: 데이터베이스 서버가 재시작되어 기존 유지되던 TCP 세션들이 만료됨
- When: 어플리케이션이 다시 API 요청을 받아 질의함
- Then: `pool_pre_ping=True` 설정에 의해 앱이 죽거나 Internal 에러 없이 새로운 세션을 스스로 재수립하여 정상 응답한다.

## ⚙️ Technical Constraints
- 동기 구문(`SessionLocal`, `psycopg2`) 사용 절대 지양. FastAPI 아키텍처에 맞춘 100% Async 기반.

## 🏁 Definition of Done (DoD)
- [ ] `database.py` 소스 코드 스크립트 작성 완료 (비동기 엔진)?
- [ ] DB 장애 시 Rollback 방어가 제너레이터 레벨에 삽입 완료되었나?
- [ ] RPO/RTO 보장용 클라우드 (Supabase) 백업 가이드가 문서화 되었는가?
