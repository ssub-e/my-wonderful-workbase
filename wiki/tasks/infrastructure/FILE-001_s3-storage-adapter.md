---
name: Feature Task
about: SRS 기반의 구체적인 개발 태스크 명세
title: "[Feature] FILE-001: S3 연동 테넌트 로고 및 사용자 프로필 이미지 업로드 어댑터"
labels: 'feature, backend, infra, priority:medium'
assignees: ''
type: task
tags: [task, infrastructure]
created: 2026-04-22
---
## 🎯 Summary
- 기능명: [FILE-001] AWS S3/Supabase Storage 기반 파일 관리 어댑터
- 목적: C-TEC-011 구현. 화주사 커스텀 브랜딩을 위한 로고 파일, PDF 리포트에 삽입할 인장 이미지, 그리고 사용자 프로필 아바타를 영속적으로 보관하고 안전하게 서빙(Presigned URL)하기 위한 통합 스토리지 미들웨어를 구축한다.

## 🔗 References (Spec & Context)
> 💡 AI Agent & Dev Note: 작업 시작 전 아래 문서를 반드시 먼저 Read/Evaluate 할 것.
- 인프라 제약: `C-TEC-011` (S3 호환 스토리지)
- 보안 가이드: `SEC-001` (민감 정보 보호)

## ✅ Task Breakdown (실행 계획)
- [ ] `app/adapters/storage_adapter.py`: `boto3` 또는 `aiobotocore` 기반 S3 연결 클래스 작성
- [ ] `UPLOAD_HISTORY` 테이블 설계: id, tenant_id, file_path, original_name, file_size, content_type
- [ ] 핵심 메서드 구현:
  - `upload_file(file, bucket, path)` : 유효성 검사 후 업로드
  - `get_signed_url(path, expires)` : 전용 다운로드 링크(Presigned) 생성
  - `delete_file(path)` : 물리적 삭제 로직
- [ ] 이미지 업로드 시 보안 검사: 허용된 확장자(JPG, PNG, SVG) 및 용량(max 2MB) 제한
- [ ] 테넌트 격리: S3 구조를 `/{tenant_id}/logos/`, `/{tenant_id}/profiles/` 와 같이 구성하여 교차 접근 방지

## 🧪 Acceptance Criteria (BDD/GWT)
Scenario 1: 테넌트 로고 업로드 및 조회
- Given: 관리자가 '우리 회사 로고' 파일을 선택함
- When: 업로드를 요청함
- Then: 파일명은 UUID로 난독화되어 S3에 저장되고, DB에는 원본 파일명과 S3 경로가 기록된다. 대시보드에서는 Presigned URL을 통해 로고가 정상 출력된다.

Scenario 2: 타사 파일 접근 시도 차단
- Given: 테넌트 A의 유저가 테넌트 B의 프로필 이미지 경로(`/{tenant_B}/profiles/x.jpg`)를 알게 됨
- When: 직접 S3 URL로 접근하거나 백엔드에 서명을 요청함
- Then: 백엔드 어댑터에서 `tenant_id` 불일치를 감지하여 `403 Forbidden` 처리하거나 서명 발급을 거부한다.

## ⚙️ Technical & Non-Functional Constraints
- 성능: 업로드 직후 사용자 화면에 반영되는 속도 ≤ 1초
- 운영: 스토리지 비용 절감을 위한 로고 파일 이미지 압축(WebP 변환) 처리 로직 검토
- 안정성: 스토리지 장애 시 폴북 처리(Default Logo 노출) 및 Slack 알림 연동

## 💻 Implementation Snippet (Storage Adapter)
```python
import aiobotocore

class S3StorageAdapter:
    def __init__(self, settings):
        self.session = aiobotocore.get_session()
        self.config = {
            'aws_access_key_id': settings.S3_KEY,
            'aws_secret_access_key': settings.S3_SECRET,
            'endpoint_url': settings.S3_ENDPOINT
        }

    async def upload_public_image(self, file_data: bytes, path: str, content_type: str):
        """파일 업로드 및 결과 반환"""
        async with self.session.create_client('s3', **self.config) as s3:
            await s3.put_object(
                Bucket='antigravity-assets',
                Key=path,
                Body=file_data,
                ContentType=content_type
            )
```

## 🏁 Definition of Done (DoD)
- [ ] 모든 Acceptance Criteria를 충족하는가?
- [ ] 2MB 이상의 대형 파일 업로드 시 `413 Request Entity Too Large` 에러가 발생하는가?
- [ ] 업로드된 이미지 파일이 서버 메모리에 남지 않고 스트리밍 방식으로 전송되는가?

## 🚧 Dependencies & Blockers
- Depends on: `INFRA-001` (DB 연결)
- Blocks: `ADMIN-009`, `VIEW-022`
