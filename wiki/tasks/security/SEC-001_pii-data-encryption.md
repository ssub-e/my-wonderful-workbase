---
name: Feature Task
about: SRS 기반의 구체적인 개발 태스크 명세
title: "[Compliance] SEC-001: 개인정보(PII) 양방향 암호화 처리 미들웨어"
labels: 'backend, security, compliance, phase:8, priority:critical'
assignees: ''
type: task
tags: [task, security]
created: 2026-04-22
---
## 🎯 Summary
- 기능명: [SEC-001] 고객/알바 구직자 연락처 등 개인식별정보(PII) DB 암호화
- 목적: B2B 서비스를 운영하다 DB가 통채로 클라우드에서 유출되는 사태가 벌어지더라도 화주사의 소비자 이름, 휴대폰 번호가 평문(Plain-text)으로 돌아다니지 못하게 막는다. ISO 27001 및 정보통신망법 준수를 위한 필수 컴플라이언스 요건.

## 🔗 References (Spec & Context)
- 기반 기술: `cryptography.fernet` 라이브러리
- 환경 변수: `DEPLOY-003` (`FERNET_KEY`)

## ✅ Task Breakdown (실행 계획)

### 1. 양방향 대칭 암복호화 유틸리티
- [ ] `app/core/security/encryption.py`:
  ```python
  from cryptography.fernet import Fernet
  from app.core.config import settings

  # settings.FERNET_KEY 는 반드시 32 url-safe base64-encoded bytes 여야함
  cipher_suite = Fernet(settings.FERNET_KEY.encode())

  def encrypt_pii(text: str) -> str:
      if not text:
          return text
      return cipher_suite.encrypt(text.encode()).decode()

  def decrypt_pii(encrypted_text: str) -> str:
      if not encrypted_text:
          return encrypted_text
      try:
          return cipher_suite.decrypt(encrypted_text.encode()).decode()
      except Exception:
          return encrypted_text # 복호화 실패 시 원시 반환(구버전 호환)
  ```

### 2. SQLModel 수명주기 훅 혹은 Getter/Setter 적용 
- [ ] `app/domain/order/model.py`:
  - `customer_phone`, `customer_name` 과 같은 필드가 DB로 `insert` 되거나 `update` 될 때 Repository 단에서 무조건 `encrypt_pii()` 를 거쳐 바이너리 해싱 텍스트(`gAAAAAB...`) 형태로 저장되게 만든다.
  - 반대로 클라이언트(대시보드)로 쏠 때는 읽어서 `decrypt_pii` 처리 후 `*` 마스킹(010-1234-****)까지 씌워 응답 객체(Dto)를 조립해준다.

## 🧪 Acceptance Criteria (BDD/GWT)

**Scenario 1: Database Raw Query 열람 시 암호화 방어 검증**
- Given: 내부 담당자가 `psql` 혹은 `DBeaver` 로 프로덕션 데이터베이스 타겟 테이블을 셀렉트(`SELECT * FROM orders`) 함.
- When: `customer_name` 및 `customer_phone` 열을 육안으로 확인.
- Then: "홍길동", "010-1234-5678" 같은 평문은 절대 보이지 않고, `gAAAAABkLw_y...` 형태의 Fernet 암호화 문자열만이 적재되어 있어야 하며, 이를 통해 개인정보 유출 리스크를 완전히 차단함.

## ⚙️ Technical Constraints
- 이 방식은 암호화된 컬럼에 대해 완전 일치(=) 검색만 제한적으로 가능해질 뿐, "홍길동" 이라는 글자가 "길동" 으로 LIKE (부분매치 DB 인덱싱) 검색이 불가능해지는 부작용을 일으킨다. 이름으로 리스트를 검색해야 하는지 기획 요건과 충돌하지 않는지 설계시 반드시 짚어보고, 만약 검색이 필요하다면 해시 인덱스(Blind Index) 기법을 고민해야 한다 (MVP 단계에선 Like 검색 폐기 권장).

## 🏁 Definition of Done (DoD)
- [ ] Fernet 대칭키를 활용한 En/Decrypt 듀얼 파이프 클래스 확보?
- [ ] `customer_phone` 등 핵심 컴플라이언스 대상 PII 적재 지점 패치 유무?
