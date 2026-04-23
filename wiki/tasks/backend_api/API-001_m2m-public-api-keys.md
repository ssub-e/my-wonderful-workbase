---
name: Feature Task
about: SRS 기반의 구체적인 개발 태스크 명세
title: "[API] API-001: M2M(Server-to-Server) 외부 연동용 고정 API Key 보안 인증체계"
labels: 'backend, api, security, phase:8, priority:medium'
assignees: ''
type: task
tags: [task, backend_api]
created: 2026-04-22
---
## 🎯 Summary
- 기능명: [API-001] 시스템간 호출(M2M) 헤더 방식의 Public API Key 발급기
- 목적: B2B 시스템 답게 "저희 회사 ERP에서 바로 API로 데이터를 귀사 B2B SaaS에 넣을수 없나요?" 라는 요구사항을 수용한다. 로그인을 요구하는 JWT 대신 브라우저 밖 기계에서 영구적으로 쓸 수 있는 고정 해시를 제공하는 방식이다.

## 🔗 References (Spec & Context)
- 인증 연계망: `AUTH-001` (JWT 와 투트랙(Two-track) 으로 구동)
- 타겟 대상: `/api/v1/public/*` 네임스페이스 한정 오픈

## ✅ Task Breakdown (실행 계획)

### 1. API_KEY 스토리지 구성
- [ ] `DB` 테이블 생성: `API_KEY_STORE`
  - 필드: `id`, `tenant_id` (소유자), `hashed_api_key`(해싱), `prefix`(보여주기용 앞 4자리), `is_active`

### 2. Header `X-API-Key` 추출 의존성 구축
- [ ] `app/api/deps.py` :
  ```python
  from fastapi import Depends, HTTPException, Security
  from fastapi.security import APIKeyHeader
  from app.core.security import verify_password  # Bcrypt 검증기 재사용

  # 클라이언트가 헤더에 "X-API-Key: B2BAI-c8a32d1..." 이런식으로 던지도록 유도
  api_key_header = APIKeyHeader(name="X-API-Key", auto_error=False)

  def get_tenant_by_api_key(api_key: str = Security(api_key_header), db=Depends(get_db)):
      if not api_key:
          raise HTTPException(status_code=401, detail="API Key 가 누락되었습니다.")
      
      # 1. API Key 원문을 앞 4~8자리 Prefix로 DB에서 대역 탐색
      prefix = parse_prefix(api_key)
      key_record = db.query(ApiKeyStore).filter(ApiKeyStore.prefix == prefix, ApiKeyStore.is_active == True).first()
      
      if not key_record:
          raise HTTPException(status_code=403, detail="등록되지 않은 키 구조입니다.")
          
      # 2. Hash 대조 (비밀번호 검증 로직과 동일)
      if not verify_password(api_key, key_record.hashed_api_key):
          raise HTTPException(status_code=403, detail="유효하지 않은 API Key 입니다.")
      
      return key_record.tenant_id
  ```

## 🧪 Acceptance Criteria (BDD/GWT)

**Scenario 1: 평문 키 유출 사고 시 DB 안전성 통제**
- Given: 해커가 SaaS 서비스 내부망 DB를 다운프을 떴음
- When: `API_KEY_STORE` 테이블을 열어 화주사들의 통신 권한을 훔치려 함
- Then: 테이블 안의 데이터는 평문(`B2BAI-xxx`)이 아니라 암호화 해싱 알고리즘(`$2b$12$...` Bcrypt)으로 뭉개져 있으므로 역산이 불가능하며 해커는 아무런 권한 침투도 행할 수 없다 (단순 토큰 저장 방식의 비극 예방).

## ⚙️ Technical Constraints
- M2M API는 브라우저가 아니므로 CORS 정책에 프리플라이트(Pre-flight)를 막거나, Rate Limit(초당 N회)를 JWT보다 더욱 보수적으로 잡아 기계의 "무한 루프(While true) 어택"에 서버가 뻗는 사태를 필히 방어한다.

## 🏁 Definition of Done (DoD)
- [ ] Header 에서 `X-API-Key` 스트링을 잡아당기는 검사 미들웨어 수립 여부?
- [ ] 발급시 원문을 1회만 보여주고 DB에는 Hash 로만 남기는 보안 단방향 룰 성립?
