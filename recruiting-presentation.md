---
marp: true
theme: gaia
class: lead
paginate: true
backgroundColor: #fff
backgroundImage: url('https://marp.app/assets/hero-background.jpg')
header: 'DeepDemand AI Demand Forecasting SaaS Recruiting'
footer: '© 2026 DeepDemand AI SaaS Project'
---

# **함께 세상을 예측할 팀원을 찾습니다**
### B2B 수요예측 AI SaaS: 실무자를 위한 "결재 방어" 엔진

**Presented by: Project Founder**

---

## **1. 우리가 해결하려는 고통 (The Problem)**

- **MD의 매일 아침**: 날씨, 트렌드, 매출 데이터를 엑셀로 수기 취합 (평균 4시간 소요)
- **승인의 벽**: 통계적 근거 부족으로 인한 임원진의 결재 반려 및 감(Intuition) 기반 발주
- **비용의 누수**: 오예측으로 인한 매일 80만 원 이상의 악성 재고 폐기 및 인건비 낭비

> "기술은 화려하지만, 정작 실무자의 '내일'을 구원하는 솔루션은 없습니다."

---

## **2. 우리의 솔루션 (The Solution)**

### **"기존 시스템에 기생(Add-on)하는 AI 비서"**

1. **XAI 해설 엔진**: 단순 수치가 아닌, 상사를 설득할 수 있는 "이유 있는 리포트" 자동 생성
2. **1-Click 데이터 연동**: 카페24, 스마트스토어 API를 한 번에 연결하여 실시간 동기화
3. **인건비 옵티마이저**: 내일 필요한 알바 인원수를 역산하여 알림톡으로 전송

---

## **3. 핵심 가치 제안 (Value Proposition)**

- **MD 결재 방어 PDF**: 4시간의 작업을 10분으로 단축, 결재 반려율 0% 도전
- **재무적 ROI 중심**: 단순 % 오차율이 아닌, "이번 달 절감액 $XXX"로 가치 증명
- **No-Code/Low-Touch**: 복잡한 도입 과정 없이 기존 업무 툴(Slack, Sheet)에 즉시 이식

---

## **4. 시장의 기회 (Market Opportunity)**

- **SAM (유효 시장)**: 글로벌 SaaS형 수요예측 시장 **약 7,300억 원**
- **초기 SOM (국내 목표)**: 이커머스 SME 타겟 **연 매출 20억 원** 목표
- **Why Now?**: 외부 변수(기상, SNS 트렌드) 민감도가 극대화된 시대, 하지만 이를 처리할 데이터 팀이 없는 중소기업의 폭발적 수요

---

## **5. 기술 스택: "MVP에 미친 파이썬 단일 스택"**

우리는 오버엔지니어링을 경계하고, **빠른 실행력**에 집중합니다.

- **Backend/Frontend**: FastAPI + Streamlit (Python 100%)
- **AI/ML**: LightGBM, Prophet, SHAP (XAI Core)
- **LLM**: Google Gemini API (비즈니스 톤 리포트 생성)
- **Infra**: Docker, Railway, PostgreSQL (SQLModel)

---

## **6. 우리의 아키텍처 (Tech Edge)**

- **Application Level Isolation**: `tenant_id` 기반의 철저한 멀티테넌트 격리
- **Internal Pipeline**: 외부 API 장애를 대비한 Redis 기반 Fallback 전략
- **Smart Scheduling**: APScheduler를 활용한 자동 데이터 수집 및 알림 전송 시스템

---

## **7. 개발 마일스톤 (Roadmap)**

- **Phase 1 (현재)**: 코어 엔진 개발 및 146개 태스크 기반 인프라 구축 완료
- **Phase 2 (Next)**: 리테일/이커머스 SME 대상 베타 테스트 및 1-Click API 모듈 고도화
- **Phase 3 (Scale-up)**: 제조/물류 중견기업용 프라이빗 엣지 환경 및 Airflow 도입

---

## **8. 우리가 찾는 동료 (Who We Need)**

- **Backend Engineer**: FastAPI와 SQLModel로 탄탄한 SaaS 뼈대를 세우실 분
- **AI/ML Engineer**: SHAP과 LLM을 융합해 "말이 통하는 AI"를 만드실 분
- **Growth Hacker**: 이커머스 MD와 센터장의 고통을 비즈니스 언어로 풀어나가실 분

---

## **9. 우리의 문화 (Culture & Vision)**

- **Vibe Coding**: AI 에이전트를 적극적으로 활용해 개발 생산성을 극대화합니다.
- **Deep Context**: 위키 기반의 지식 공유로 모든 팀원이 비즈니스와 기술을 100% 이해합니다.
- **Add-on Strategy**: 무거운 플랫폼을 만들기보다, 고객의 업무 흐름에 자연스럽게 스며듭니다.

---

## **10. Join the Vision!**

### **"실무자의 사내 정치를 방어하고, 재고 누수를 막는 AI의 힘"**

우리는 단순한 예측 모델을 넘어, **실제 의사결정의 혁신**을 만듭니다.
이 여정에 함께하여 초기 멤버로서 SaaS의 근간을 설계해 주세요.

- **Contact**: [Your Email/Link]
- **Wiki**: `wiki/recruiting-presentation.md`

---
