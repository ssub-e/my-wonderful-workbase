---
name: Feature Task
about: SRS 기반의 구체적인 개발 태스크 명세
title: "[Worker] F3-W03: 알림톡 및 인원 역산 결과 송출 자동화 워커"
labels: 'worker, notification, phase:1, priority:high'
assignees: ''
type: task
tags: [task, misc]
created: 2026-04-22
---
## 🎯 Summary
- 기능명: [F3-W03] 카카오 알림톡 타겟 추출 및 전송 Worker
- 목적: 매일 16:00 (물류 관리자 퇴근 시점 전)에 스케줄러로 발동. 당일 도출된 `WORKFORCE_PLAN`(예측 인원 역산치 데이터)을 DB에서 읽어, 테넌트 관리자의 SMS/카카오톡으로 자동 푸시 발송한다. 발송 이후 성공/실패 여부를 Audit과 Plan 테이블에 찍어 남긴다.

## 🔗 References (Spec & Context)
- SRS 요구사항: [`SRS-V1.0.md#§4.1.3`](file:///e:/workspace/SRS-from-PRD/SRS-V1.0.md) — REQ-FUNC-022(16시 자동 알림 발송), REQ-FUNC-024(발송 이력 추적)
- 연관 태스크: `ADAPT-004` (카카오/SMS 폴백 송신 프로토콜), `DB-009` (Workforce 모델 저장소)

## ✅ Task Breakdown (실행 계획)

### 1. 전송 결합 워커 작성
- [ ] `app/domain/workforce/workers/alimtalk_worker.py`:
  ```python
  import logging
  from datetime import date
  from sqlalchemy.ext.asyncio import AsyncSession
  from app.core.retry_handler import with_etl_retry
  
  logger = logging.getLogger(__name__)

  @with_etl_retry(max_retries=2, base_delay_sec=10) # 통신 에러 방어
  async def run_alimtalk_sender_worker():
      target_date = date.today()
      logger.info(f"Triggering 16:00 Alimtalk notifications for {target_date} plans")
      
      async with AsyncSessionLocal() as session:
          # 1. 당일 역산된 플랜 중 미발송(`notified_at` IS NULL) 인 항목 추출
          pending_plans = await get_plans_pending_notification(session, target_date)
          
          # 2. 어댑터 초기화
          adapter = KakaoAlimtalkAdapter(api_key=settings.KAKAO_KEY, sender_key=settings.KAKAO_SENDER_KEY)
          
          for plan in pending_plans:
              tenant_info = await get_tenant_contact_info(session, plan.tenant_id)
              
              try:
                  # 3. 전송 시도 API (SMS Fallback 로직은 어댑터 단에서 감춰져 처리됨)
                  is_success = await adapter.send_workforce_plan(
                      phone_number=tenant_info.admin_phone,
                      center_name=plan.center_id,
                      recommended_qty=plan.recommended_workers,
                      date=(target_date + timedelta(days=1)).strftime("%m월 %d일")
                  )
                  
                  # 4. 성공 시 타임스탬프 기록
                  if is_success:
                      await update_notified_status(session, plan.id, "kakaotalk_or_sms")
                      logger.info(f"Plan [{plan.id}] - Notification sent to Tenant {plan.tenant_id}")
                      
              except Exception as e:
                  logger.error(f"Notification error for Plan [{plan.id}]: {e}")
                  # 다중 유저 발송 시 1인 서버 에러로 배치가 죽지 않도록 Catch & Continue 처리
                  continue
                  
          await session.commit()
  ```

## 🧪 Acceptance Criteria (BDD/GWT)

**Scenario 1: 미발송 대상 분리 (중복 발송 금지)**
- Given: 재시작된 스케줄러나 수동 발송 버튼이 실수로 16:05에 한 번 더 눌렸음
- When: `run_alimtalk_sender_worker` 내의 `get_plans_pending_notification` 질의가 수행됨
- Then: DB 쿼리상 `notified_at IS NULL` 인 대상만 선별하므로, 이미 16:00 정각에 전송되어 스탬프가 찍혀있는 Row 들은 쿼리 결과망에서 걸러지고, 중복된 센터장 카카오톡 알림톡 발송 공해가 원천적으로 방어된다.

**Scenario 2: Fallback 무결성 지원 및 로깅**
- Given: 카카오 망 연동 벤더사 응답 지연 타임아웃
- When: 어댑터 내부에 숨겨진 `Fallback SMS` 가 대신 쏴져서 True를 최종 반환합
- Then: 워커 서비스 레이어는 이를 단순히 True(통신 성공) 로 인식하여 `notified_at` 상태 스탬핑을 정상 강행하여 배치 워커의 독립성과 결합도를 깔끔하게 유지한다.

## ⚙️ Technical Constraints
- 발송 간 지연(Delay): B2B 수 백명 규모 전송 시에 `for` 루프를 순회하며 쏘다가 카카오 측의 초당 전송 리미트(TPS) 초과 차단을 방지해야 한다. `asyncio.sleep(0.1)` 류의 미세 트래픽 컨트롤 로직 포함이 권장됨.
- 예외 전파 유무: 1건 수집 실패는 전체 Exception 처리를 해야하지만, 1건 발송 실패는 해당 에러만 무시(Continue)하고 다음 사장님(Tenant) 에게 진행되도록 설계한다.

## 🏁 Definition of Done (DoD)
- [ ] 미발송 내역(`notified_at IS NULL`)만 필터 조달하는 DB Fetch 쿼리 연결?
- [ ] 오류 시에도 타겟 루프가 멈추지 않는 `try/except/continue` 블럭 적용 완정?
- [ ] 초당 발송량(TPS) 완화 코드 라인 구성?
