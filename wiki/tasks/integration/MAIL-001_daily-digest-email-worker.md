---
name: Feature Task
about: SRS 기반의 구체적인 개발 태스크 명세
title: "[Feature] MAIL-001: 매일 오전 'Daily Digest' 요약 리포트 메일 워커 및 템플릿"
labels: 'feature, backend, worker, priority:medium'
assignees: ''
type: task
tags: [task, integration]
created: 2026-04-22
---
## 🎯 Summary
- 기능명: [MAIL-001] 출근 전 주요 예측 현황 요약 메일(Daily Digest) 발송기
- 목적: REQ-FUNC-022 확장. 바쁜 MD들이 아침에 대시보드에 접속하기 전, 간밤에 집계된 익일 예측 물동량의 급증/급락 포인트와 주요 기상 변수를 이메일 한 통으로 요약하여 송신함으로써 업무의 반응 속도를 높인다.

## 🔗 References (Spec & Context)
> 💡 AI Agent & Dev Note: 작업 시작 전 아래 문서를 반드시 먼저 Read/Evaluate 할 것.
- 스케줄러 인프라: `INFRA-003` (APScheduler)
- 메일 엔진 어댑터: `ADAPT-004` (SendGrid 또는 SMTP 확장)
- 예측 데이터: `FORECAST`

## ✅ Task Breakdown (실행 계획)
- [ ] `SUMMARY_MAIL_LOG` 테이블 설계: id, tenant_id, sent_at, status, recipient_count
- [ ] Jinja2 기반의 이메일 HTML 템플릿 작성 (브랜딩 컬러 적용, 시계열 요약표 포함)
- [ ] `DailyDigestWorker` 신설: 매일 오전 08:30(KST) 트리거
- [ ] 집계 로직: 
  - 최근 24시간 내 예측된 물동량 중 변동폭이 가장 큰 SKU Top 3 추출
  - 오늘/내일 강수 확률 및 기온 특이점 요약
  - 미결제(Overdue) 등 계정 경고 상태 포함
- [ ] 대량 발송 최적화: `tenant_id`별 유저 목록을 돌며 비동기 발송 (`AnyIO` 태스크 그룹 활용)

## 🧪 Acceptance Criteria (BDD/GWT)
Scenario 1: 정상적인 요약 메일 수신
- Given: MD가 매일 오전 메일링 서비스를 구독 중인 상태
- When: 08:30에 스케줄러가 작동함
- Then: MD의 이메일함에 "[(주)화주사] 오늘 물류 현황 요약: A 상품 20% 증가 예상" 이라는 제목의 메일이 도착하며, 내부에는 깔끔한 HTML 요약표가 렌더링되어 있다.

Scenario 2: 데이터 부재 시 발송 생략
- Given: 연동 오류로 어제 예측 데이터가 전혀 생성되지 않은 테넌트
- When: 워커가 실행됨
- Then: 의미 없는 빈 메일을 보내지 않고 `SKIP` 로그를 남기며, 운영팀에 해당 테넌트의 데이터 단절 사실을 통보한다.

## ⚙️ Technical & Non-Functional Constraints
- 성능: 전사 유저 메일 발송 완료 시간 ≤ 15분 (병렬 처리 필수)
- 호환성: Outlook, Gmail 등 다양한 메일 클라이언트에서 레이아웃이 깨지지 않는 Inline CSS 적용
- 신뢰성: 발송 실패 시 1회 재시도 로직(`ETL-002` 재활용)

## 💻 Implementation Snippet (Email Service)
```python
from jinja2 import Environment, FileSystemLoader

async def send_daily_digest(tenant_id: uuid.UUID, users: List[User], stats: dict):
    env = Environment(loader=FileSystemLoader("app/templates/emails"))
    template = env.get_template("daily_digest.html")
    
    # 이메일 내용 렌더링
    html_content = template.render(
        company_name=stats['company_name'],
        top_skus=stats['top_skus'],
        weather_alert=stats['weather_alert'],
        dashboard_url=f"{settings.BASE_URL}/dashboard"
    )
    
    # 비동기 발송 위임
    async with anyio.create_task_group() as tg:
        for user in users:
            tg.start_soon(
                email_adapter.send,
                user.email,
                f"[{stats['company_name']}] 오늘 물류 현황 요약",
                html_content
            )
```

## 🏁 Definition of Done (DoD)
- [ ] 모든 Acceptance Criteria를 충족하는가?
- [ ] 메일 템플릿 내의 링크가 실제 대시보드 URL과 정확히 연결되는가?
- [ ] `Accept-Language` 헤더에 따른 메일 내용 다국어(`I18N-002`) 처리 여부 확인?

## 🚧 Dependencies & Blockers
- Depends on: `ADAPT-004`, `F1-003` (예측 완료 데이터)
- Blocks: 없음
