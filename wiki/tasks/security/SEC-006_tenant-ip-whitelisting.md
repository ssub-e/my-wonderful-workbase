---
name: Feature Task
about: SRS 기반의 구체적인 개발 태스크 명세
title: "[Feature] SEC-006: 특정 기업용 '접속 허용 IP 화이트리스트(Whitelisting)' 보안 엔진"
labels: 'feature, backend, security, priority:medium'
assignees: ''
type: task
tags: [task, security]
created: 2026-04-22
---
## 🎯 Summary
- 기능명: [SEC-006] 테넌트별 접속 IP 대역 제어 시스템
- 목적: 보안 정책이 엄격한 대형 엔터프라이즈 화주사가 자사의 사무실 IP나 전용선 IP 대역에서만 시스템 접속이 가능하도록 제한하는 기능을 제공한다. 이는 계정 정보가 유출되더라도 외부에서의 무단 접속을 원천 차단하여 기업 데이터를 보호하기 위함이다.

## 🔗 References (Spec & Context)
> 💡 AI Agent & Dev Note: 작업 시작 전 아래 문서를 반드시 먼저 Read/Evaluate 할 것.
- 가용성 정책: REQ-NF-017 (강력한 테넌트 고립)
- 관련 태스크: `SEC-005` (Rate Limiting 미들웨어)

## ✅ Task Breakdown (실행 계획)
- [ ] `TENANT_IP_WHITELIST` 테이블 설계: id, tenant_id, ip_address(CIDR 포맷 지원), description, created_at
- [ ] **IP Validation Middleware** 구현:
  - 모든 API 요청 또는 로그인 요청 시 클라이언트의 `X-Forwarded-For` 또는 `Remote-Addr` 헤더 추출
  - 해당 테넌트의 화이트리스트 설정 유무 확인
  - 설정이 있다면, 현재 IP가 허용된 CIDR 범위(`ipaddress` 라이브러리 활용)에 속하는지 검증
- [ ] **Bypass Policy**: `SUPER_ADMIN` 계정 또는 글로벌 화이트리스트(재난 복구용 등) 예외 처리 로직
- [ ] 관리자 페이지(`ADMIN-001`) 내 IP 등록/삭제 UI와 연동되는 CRUD API 구축
- [ ] 비정상 IP 접속 시도 시 `AUDIT_LOG` 에 'SECURITY_ALERT' 로깅 및 알림톡 발송

## 🧪 Acceptance Criteria (BDD/GWT)
Scenario 1: 허용된 IP에서의 접속
- Given: 테넌트 A의 화이트리스트에 `123.123.123.0/24` 가 등록됨
- When: 유저가 `123.123.123.45` IP로 로그인을 시도함
- Then: 정상적으로 로그인 프로세스가 진행된다.

Scenario 2: 허용되지 않은 IP에서의 차단
- Given: 위와 동일한 설정
- When: 유저가 카페나 자택(IP: `211.211.211.10`)에서 접속을 시도함
- Then: 서버는 `403 Forbidden` 과 함께 "허용되지 않은 IP에서의 접근입니다. 관리자에게 문의하세요." 메시지를 반환한다.

## ⚙️ Technical & Non-Functional Constraints
- 성능: 매 요청마다 DB 조회를 피하기 위해 화이트리스트 목록을 Redis(`INFRA-002`)에 캐싱
- 정확성: `X-Forwarded-For` 변조 공격(Spoofing)을 막기 위해 신뢰할 수 있는 프록시(Nginx/Cloudflare) 설정 확인 필수
- 안정성: 관리자가 실수로 자신의 현재 IP를 제외하고 등록하여 락인(Lock-in)되는 상황 방지를 위해, 등록 시 '현재 IP 포함 여부' 체크 경고 모달 제공

## 💻 Implementation Snippet (CIDR Validation Logic)
```python
import ipaddress

def is_ip_allowed(client_ip: str, allowed_cidrs: List[str]) -> bool:
    """
    클라이언트 IP가 허용된 CIDR 대역 중 하나에 포함되는지 확인
    """
    if not allowed_cidrs:
        return True # 설정이 없으면 전체 허용
        
    client_addr = ipaddress.ip_address(client_ip)
    for cidr in allowed_cidrs:
        if client_addr in ipaddress.ip_network(cidr):
            return True
    return False

# In FastAPI Middleware
if not is_ip_allowed(request.client.host, tenant_whitelist):
    raise HTTPException(status_code=403, detail="IP_NOT_ALLOWED")
```

## 🏁 Definition of Done (DoD)
- [ ] 모든 Acceptance Criteria를 충족하는가?
- [ ] IPv4 및 IPv6 대역을 모두 지원하는가?
- [ ] Redis 캐시 무효화(Cache Invalidation) 로직이 IP 목록 수정 시 즉시 반영되는가?

## 🚧 Dependencies & Blockers
- Depends on: `INFRA-002`, `SEC-005`
- Blocks: 없음
