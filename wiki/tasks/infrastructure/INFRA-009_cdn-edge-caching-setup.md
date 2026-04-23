---
name: Feature Task
about: SRS 기반의 구체적인 개발 태스크 명세
title: "[Feature] INFRA-009: 글로벌 정적 자산 로딩 가속을 위한 'CDN 및 에지 캐싱(Edge Caching)' 연동"
labels: 'feature, backend, infra, priority:low'
assignees: ''
type: task
tags: [task, infrastructure]
created: 2026-04-22
---
## 🎯 Summary
- 기능명: [INFRA-009] 리소스 로딩 속도 최적화를 위한 CDN 인프라 구축
- 목적: REQ-NF-001 보장. 전 세계 어디서든 대시보드와 리포트에 포함된 이미지, 로고, 정적 에셋(JS/CSS)을 지연 없이 불러올 수 있도록 AWS CloudFront 또는 Cloudflare CDN을 연입하고, 에지 서버에서 캐싱을 수행하여 원본 서버(Origin)의 부하를 획기적으로 낮춘다.

## 🔗 References (Spec & Context)
> 💡 AI Agent & Dev Note: 작업 시작 전 아래 문서를 반드시 먼저 Read/Evaluate 할 것.
- 스토리지: `FILE-001` (S3 어댑터)
- 프론트엔드 배포: `DEPLOY-001` (Vercel/Static Hosting)

## ✅ Task Breakdown (실행 계획)
- [ ] **CDN Distribution** 생성: AWS CloudFront 또는 유사 서비스로 S3 버킷(Origin) 연동
- [ ] **Cache Policy** 정의: 
  - 정적 에셋(JS/CSS/Font): 1년(`max-age=31536000`) 캐싱 및 버전 해시 사용
  - 테넌트 로고 및 이미지: 24시간 캐싱 및 변경 시 `Invalidation` 로직 연동
- [ ] **Origin Access Control (OAC)**: S3 버킷에 직접 접근을 차단하고 오직 CDN을 통해서만 자산을 서빙하도록 보안 설정
- [ ] **Custom Domain & SSL**: `static.example.com` 등의 커스텀 도메인 연결 및 HTTPS 인증서 적용
- [ ] `FILE-001` 어댑터 확장: 파일 조회 시 S3 직접 URL 대신 CDN 도메인 주소가 포함된 URL 발급

## 🧪 Acceptance Criteria (BDD/GWT)
Scenario 1: 에지 캐싱을 통한 로딩 속도 개선
- Given: 글로벌 화주사가 멀리 떨어진 리전에서 대시보드에 접속함
- When: 500KB 상당의 테넌트 로고 이미지를 불러옴
- Then: 첫 번째 요청(Miss) 이후 두 번째 요청(Hit)부터는 통신 속도가 100ms 이내로 단축되어야 하며, `X-Cache: Hit from cloudfront` 헤더가 확인되어야 한다.

Scenario 2: 로고 변경 시 즉시 갱신
- Given: 관리자가 `ADMIN-009` 를 통해 사내 로고 이미지를 교체함
- When: 교체 완료 시점
- Then: 시스템은 해당 파일 경로에 대해 CDN 캐시 무효화(`Invalidation`) 명령을 쏘고, 유저는 즉시 변경된 새 로고를 확인해야 한다.

## ⚙️ Technical & Non-Functional Constraints
- 성능: 정적 에셋의 압축 전송(Brotli/Gzip) 기능을 CDN 레벨에서 활성화
- 비용: 무분별한 캐시 무효화는 비용을 발생시키므로, 버전 관리(Versioning) 위주의 갱신 전략 수립
- 보안: CDN 헤더 조작을 통한 무단 접근 방지를 위한 'Signed URL' 연동 고려 (민감 리포트 한정)

## 💻 Implementation Snippet (CDN URL Translation Idea)
```python
class StorageAdapter:
    def get_public_url(self, file_key: str) -> str:
        """
        S3 URL을 가속된 CDN URL로 변환하여 반환
        """
        if settings.USE_CDN:
            return f"https://{settings.CDN_DOMAIN}/{file_key}"
        return f"https://{settings.S3_BUCKET}.s3.amazonaws.com/{file_key}"

# In API response
{
    "logo_url": storage.get_public_url("tenants/1/brand/logo.png")
}
```

## 🏁 Definition of Done (DoD)
- [ ] 모든 Acceptance Criteria를 충족하는가?
- [ ] CDN 적용 후 어필 로딩 타임 FCP(First Contentful Paint)가 기준치 이하로 단축되었는가?
- [ ] 보안 취약점(CORS 설정 등)이 엔터프라이즈 표준에 맞게 세팅되었는가?

## 🚧 Dependencies & Blockers
- Depends on: `FILE-001`, `DEPLOY-001`
- Blocks: 없음
