---
name: Deployment Task
about: SRS 기반 MVP 실운영 프로덕트 릴리즈 설정
title: "[Deploy] DEPLOY-003: 상용 배포용 보안 환경변수 및 Secrets 매니저 구성"
labels: 'infra, deploy, security, phase:6, priority:medium'
assignees: ''
type: task
tags: [task, infrastructure]
created: 2026-04-22
---
## 🎯 Summary
- 기능명: [DEPLOY-003] 프로덕션용 민감정보(Secrets) 암호화 주입 및 관리
- 목적: REQ-NF-021. 개발망과 프로덕션용 키를 완전히 분리한다. PostgreSQL 비밀번호, JWT 시크릿(`ENV-005`), Gemini 결제 API Key 등 외부에 털리면 기업의 생존이 위협받는 핵심 .env 값들을 GitHub 등 소스코드에서 배제하고 라이브 서버에 안전하게 밀어넣는 체계 명세.

## 🔗 References (Spec & Context)
- 기반 환경 모듈: `ENV-005` (Pydantic BaseSettings 모델)

## ✅ Task Breakdown (실행 계획)

### 1. 보안 인프라(Secrets Manager) 셋업 권고 
- [ ] MVP 단계 1안 (`.env.production` 파일 EC2 직접 전송):
  - 로컬에서만 쥐고 있는 `.env.production` 폴더를 GitHub Actions 의 Settings -> Secrets 에 그대로 Paste 하고 빌드 시점에 Echo 명령어 로 EC2 하드디스크 상단에 떨군다.
- [ ] 확장 단계 2안 (AWS Secrets Manager / Parameter Store):
  - `boto3` 라이브러리를 추가하여, `Settings` 클래스가 초기화 될 때 AWS 인스턴스 역할(IAM) 권한을 타고 JSON 비밀키 객체를 AWS로부터 끌어내어 인메모리에 박아넣는다 (하드디스크 기록 원천 차단).

### 2. 배포 필수 Key Valudation (100% 필수 요건)
- [ ] 서버 기동 시 강제 체크될 주요 환경변수 리스트:
  - `POSTGRES_USER` / `POSTGRES_PASSWORD` / `POSTGRES_HOST_PROD`
  - `REDIS_PASSWORD` / `REDIS_URL_PROD`
  - `SECRET_KEY` : JWT 토큰 해독 및 암호화용 256비트 대칭키 (절대 개발서버와 분리)
  - `FERNET_KEY` : `ADMIN-002` 카페24 토큰 복호화를 위한 별도 대칭키 (절대 노출 방지)
  - `GEMINI_API_KEY` : 과금 주체.
  - `SLACK_WEBHOOK_URL` (`ADAPT-006` 에러 연동용)

## 🧪 Acceptance Criteria (BDD/GWT)

**Scenario 1: `SECRET_KEY` 미등록시 서버 부팅 거부 (Fail-Fast)**
- Given: 신입 개발자가 `.env` 에 JWT용 `SECRET_KEY` 를 빼고 프로덕션 배포를 올려버림
- When: Gunicorn 이 컨테이너를 올릴 때 Pydantic 환경 변수가 기동함
- Then: `ENV-005` 에 정의된 Pydantic 모델의 `Field(..., min_length=32)` 유효성 검사 에러가 터지며 서버 구동 자체가 Crash 되므로, 취약한 암호키로 유저 로그인이 수용되는 심각한 인시던트를 즉시 방지한다.

## ⚙️ Technical Constraints
- EC2 루트 폴더나 Docker Hub 이미지 레이어 속 어딘가에 이 키가 묻어서 올라가지 않도록, 반드시 Dockerfile 내부 구조에서는 `.env` 를 `COPY` 하는 줄을 빼거나 `.dockerignore` 에 올려야 한다 (`docker-compose.yml` 의 `env_file` 의존성 지시자로만 주입한다).

## 🏁 Definition of Done (DoD)
- [ ] JWT/Fernet Key 등 프로덕션용 보안 암호키 발급 분리 룰이 성립되었나?
- [ ] GitHub Actions 나 수동 배포 환경에서 이 환경 변수를 찔어넣어줄 방법(Secrets 탭 추가 등)을 기획하였는가?
- [ ] `.dockerignore` 안에 `.env` 블로킹 룰이 확실히 들어있는가?
