---
name: Deployment Task
about: SRS 기반 MVP 실운영 프로덕트 릴리즈 설정
title: "[Deploy] DEPLOY-002: 백엔드(FastAPI) 실운영 프로덕션 서버 파이프라인 (Nginx/Uvicorn)"
labels: 'infra, deploy, backend, phase:6, priority:critical'
assignees: ''
type: task
tags: [task, infrastructure]
created: 2026-04-22
---
## 🎯 Summary
- 기능명: [DEPLOY-002] FastAPI 백엔드 프로덕션 서버(AWS EC2) 빌드 및 런칭
- 목적: `ENV-001`에서 만든 개발용 Docker-compose 환경을 격상하여, 실제 사용자의 트래픽을 받고 TLS(HTTPS)를 해독하며 Gunicorn-Uvicorn 멀티 워커를 구동하는 안정적인 배포망을 구성한다.

## 🔗 References (Spec & Context)
- 기반 인프라: `ENV-001` (기존의 단순 포트개방 `command: uvicorn`)
- CI/CD 전략: GitHub Actions 에서 EC2 로 SSH 배포하는 롤링 형태

## ✅ Task Breakdown (실행 계획)

### 1. Production Docker-compose 파일 파생
- [ ] `docker-compose.prod.yml`:
  ```yaml
  version: '3.8'
  services:
    api:
      build: 
        context: .
        dockerfile: Dockerfile
      restart: always
      # Gunicorn 을 워커 매니저로 두고 Uvicorn을 Worker Class로 사용해 CPU 코어수만큼 비동기 처리 극대화
      command: gunicorn app.main:app -k uvicorn.workers.UvicornWorker -w 4 --bind 0.0.0.0:8000
      env_file:
        - .env.production
      ports:
        - "8000:8000"
      logging:
        driver: "json-file"
        options:
          max-size: "10m"
          max-file: "5"
  ```

### 2. Reverse Proxy (Nginx) 및 SSL 세팅 가이드
- [ ] EC2(혹은 외부 로드밸런서) 최전방에 Nginx 데몬 설치.
- [ ] Nginx 설정 블록(`sites-available/api`) 작성:
  ```nginx
  server {
      listen 443 ssl;
      server_name api.b2bai.com;

      ssl_certificate /etc/letsencrypt/live/api.b2bai.com/fullchain.pem;
      ssl_certificate_key /etc/letsencrypt/live/api.b2bai.com/privkey.pem;

      location / {
          proxy_pass http://localhost:8000;
          proxy_set_header Host $host;
          proxy_set_header X-Real-IP $remote_addr;
          proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
          proxy_set_header X-Forwarded-Proto $scheme;
      }
  }
  ```
  - Let's Encrypt(`certbot`)로 위 와 같이 SSL 키를 심고 443 프로토콜을 받은 후 내부망 8000번포트(도커 징검다리)로 패스 해주는 설정이다.

## 🧪 Acceptance Criteria (BDD/GWT)

**Scenario 1: 멀티 워커 병렬 블로킹 회피**
- Given: 프로덕션 서버에 `-w 4` (워커 4대) 가 실행중
- When: 한 사용자가 무거운 CPU 연산(`F1-002: 모델 재학습`)을 트리거하여 1번 워커가 순간 블로킹됨
- Then: 나머지 일반적인 API 호출(`F2-W03: 주문조회` 등)은 Gunicorn이 똑똑하게 2, 3, 4번 워커로 트래픽을 우회 분산시켜줘서 전체 시스템이 마비되는 것을 막는다 (단일 uvicorn 서빙 대비 필수).

**Scenario 2: 컨테이너 무한 로그 비대화 방지**
- Given: 한달간 시스템이 돌아가 수십 GB의 감사/크롤링 Error 로그 콘솔에 찍힘.
- When: 도커 데몬 구동중 
- Then: `logging` 드라이버 옵션(`max-size: 10m`)에 의해 10MB 마다 로테이션되고 최대 5개(50MB)까지만 보존, 디스크가 가득 차버리는 서비스 파괴(Disk Full Outage)를 방어한다.

## ⚙️ Technical Constraints
- ASGI 환경(FastAPI)과 연동되는 Gunicorn 구동이므로 `uvicorn.workers.UvicornWorker` 모듈을 명시적으로 적어야 WSGI 에러 폭탄을 피할 수 있다.

## 🏁 Definition of Done (DoD)
- [ ] `gunicorn` 과 Uvicorn worker가 결합된 `command` 구동 스크립트 작성?
- [ ] 도커 컨테이너 로테이션 단위(Log Limit) 설정 확인?
- [ ] Nginx Reverse Proxy 와 `X-Forwarded-Proto` 헤더 포워딩 설정 명세?
