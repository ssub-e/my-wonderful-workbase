---
name: Feature Task
about: SRS 기반의 구체적인 개발 태스크 명세
title: "[Infra] ENV-003: CI/CD 파이프라인 구성 (GitHub → 클라우드 배포)"
labels: 'infra, devops, priority:high, phase:0'
assignees: ''
type: task
tags: [task, infrastructure]
created: 2026-04-22
---
## 🎯 Summary
- 기능명: [ENV-003] CI/CD 파이프라인 구성 및 GitHub Actions
- 목적: GitHub 코드 푸시 시 `Ruff` 린트와 `Pytest`가 자동 수행되어 코드 품질(REQ-NF-028 80% 이상의 커버리지)을 강제하고, main 브랜치 머지 시 Railway/Render 등의 프로덕션 환경에 자동 배포되도록 무중단 파이프라인을 구축한다 (C-TEC-010).

## 🔗 References (Spec & Context)
> 💡 AI Agent Notes:
- SRS 배포 인프라: [`SRS-V1.0.md#§1.5.3`](raw/assets/SRS-V1.0.md) — C-TEC-010 (Railway / Render 자동 배포)
- SRS 성능/검증: [`SRS-V1.0.md#§4.2.6`](raw/assets/SRS-V1.0.md) — REQ-NF-028 (테스트 커버리지 80% CI 파이프라인 통합)

## ✅ Task Breakdown (실행 계획)

### 1. CI 파이프라인 (GitHub Actions)
- [ ] `.github/workflows/ci.yml` 스크립트 작성:
  ```yaml
  name: CI Pipeline
  on:
    pull_request:
      branches: [ main ]
    push:
      branches: [ main ]
      
  jobs:
    test:
      runs-on: ubuntu-latest
      steps:
        - uses: actions/checkout@v3
        - name: Set up Python 3.11
          uses: actions/setup-python@v4
          with:
            python-version: "3.11"
            cache: 'pip'
        - name: Install dependencies
          run: |
            python -m pip install --upgrade pip
            pip install -r requirements.txt
            pip install pytest pytest-cov ruff
        - name: Lint with Ruff
          run: ruff check .
        - name: Test with pytest
          run: pytest --cov=app --cov-report=xml --cov-fail-under=80
  ```

### 2. CD 파이프라인 연동
- [ ] `main` 브랜치에 CI 성공 시 동작할 Deploy 잡 추가 (Railway 연동 예시):
  ```yaml
    deploy:
      needs: test
      if: github.ref == 'refs/heads/main'
      runs-on: ubuntu-latest
      steps:
        - uses: actions/checkout@v3
        - name: Deploy to Railway
          uses: bervProject/railway-deploy@main
          with:
            railway_token: ${{ secrets.RAILWAY_TOKEN }}
            service: "api-service"
  ```
- [ ] Railway/Render 환경을 위한 `Dockerfile` (프로덕션 릴리즈용 레이어 캐싱 최적화 버젼) 작성 확립. `ENV-001`을 재활용하되 Uvicorn Worker 세팅 포함 구동 커맨드.

### 3. Repository Secrets 문서화
- [ ] `README.md` 에 배포 파이프라인 생성을 위해 개발자가 세팅해야 할 GitHub Secrets 목록 명세 (`RAILWAY_TOKEN`, `DATABASE_URL`, `JWT_SECRET_KEY` 등)

## 🧪 Acceptance Criteria (BDD/GWT)

**Scenario 1: PR 생성 및 Linter 통과 (방어 조치)**
- Given: 의도적인 문법 에러나 미사용 import가 포함된 코드가 PR로 올라옴
- When: GitHub Actions CI가 트리거됨
- Then: `Ruff` 린터 단계에서 파이프라인이 즉각 실패(❌)하고, 머지를 막아야 함.

**Scenario 2: 테스트 커버리지 기준 (REQ-NF-028)**
- Given: 전체 애플리케이션의 테스트 코드가 부족하여 커버리지가 75%인 상태
- When: CI 의 `pytest --cov-fail-under=80` 스텝이 실행됨
- Then: 커버리지 부족을 이유로 CI가 실패(❌) 처리 되어 머지 불가 상태가 된다.

**Scenario 3: 배포 CD 성공**
- Given: CI를 통과하고 `main` 브랜치에 머지 발생
- When: CD Deploy 잡이 트리거됨
- Then: 설정된 클라우드(Railway/Render)로 컨테이너 이미지가 올라가고 서비스 다운타임 없이 신규 리비전이 구동된다.

## ⚙️ Technical Constraints
- 라이브러리: Linter/Formatter 로는 빠르고 현대적인 `Ruff`를 표준으로, 테스트는 `pytest` 와 `pytest-cov`를 결합하여 사용한다.
- 보안: 배포 토큰은 레포지토리 설정의 Secrets에 보관.

## 🏁 Definition of Done (DoD)
- [ ] `.github/workflows/ci.yml` 작성이 완료되고 PR 생성시 동작을 확인했는가?
- [ ] `--cov-fail-under=80` 조건이 파이프라인 내에 강제 적용되어 있는가?
- [ ] 클라우드 배포 스크립트 작성 및 배포 성공 내역이 존재하는가?
