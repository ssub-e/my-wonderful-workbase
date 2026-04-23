---
name: Feature Task
about: SRS 기반의 구체적인 개발 태스크 명세
title: "[Feature] VIEW-027: SKU 및 고객사 속성 확장을 위한 '사용자 정의 필드(UDF)' 관리 UI"
labels: 'feature, frontend, ux, priority:low'
assignees: ''
type: task
tags: [task, frontend]
created: 2026-04-22
---
## 🎯 Summary
- 기능명: [VIEW-027] 비즈니스 특화 정보 관리를 위한 커스텀 데이터 필드 설정
- 목적: 기 구축된 `PRODUCT`나 `SHOP` 테이블 외에, 화주사가 자사만의 특별한 속성(예: '냉동 온도 기준', '자체 등급', 'VIP 여부')을 추가하여 관리하고 싶어 하는 요구를 수용한다. DB 스키마를 직접 변경하지 않고도 유저가 필드를 동적으로 추기하고 관리할 수 있는 메타데이터 관리 시스템을 제공한다.

## 🔗 References (Spec & Context)
> 💡 AI Agent & Dev Note: 작업 시작 전 아래 문서를 반드시 먼저 Read/Evaluate 할 것.
- 데이터 모델: `PRODUCT` 테이블 내 `extra_data` (JSONB) 컬럼 활용
- 공통 컴포넌트: `VIEW-002` (Input, Badge)

## ✅ Task Breakdown (실행 계획)
- [ ] **UDF Definition UI**: 특정 엔티티(Product/Shop 등)에 추가할 필드명, 타입(Text/Number/Select), 기본값 정의 창 구현
- [ ] **Dynamic Form Renderer**: 정의된 UDF 명세에 따라 상품 수정 페이지 하단에 동적으로 인력 폼(Input) 생성 및 바인딩
- [ ] **Metadata API**: `POST /api/v1/udf/definitions` (필드 정의 저장 및 관리)
- [ ] **Validation Layer**: 정의된 타입에 맞지 않는 데이터 입력을 프론트/백엔드 양단에서 방어
- [ ] **Search Expansion**: 구축된 UDF 키를 검색 필터(`VIEW-024`)에 포함하여 동적 쿼리 생성 지원

## 🧪 Acceptance Criteria (BDD/GWT)
Scenario 1: 새로운 상품 속성 추가
- Given: 설정 페이지 내 UDF 관리 탭
- When: 상품 엔티티에 '보관 온도' (Number 타입) 필드를 정의하고 저장함
- Then: 상품 상세 페이지 진입 시 하단에 '보관 온도' 입력칸이 나타나며, 저장 시 DB의 `extra_data` 필드 내에 JSON 형태로 저장된다.

Scenario 2: 커스텀 필드 기반 검색
- Given: '냉동' 이라는 사용자 정의 태그를 가진 상품 데이터
- When: 필터에서 '냉동' 태그 검색 시도
- Then: JSONB 검색 기능을 통해 해당 태그를 가진 상품들만 정확히 추출되어야 한다.

## ⚙️ Technical & Non-Functional Constraints
- 확장성: PostgreSQL `JSONB` 인덱싱(`GIN Index`)을 활용하여 수십만 건의 커스텀 필드 검색 성능 보장
- 범용성: 필드 타입은 Text뿐 아니라 'Single Select', 'Multi Select', 'Date' 등 5가지 이상 지원
- UX: 필드 정의가 많아질 경우 섹션별 그룹화 기능 제공

## 💻 Implementation Snippet (Frontend Dynamic Field Injection)
```tsx
const UdfFieldRenderer = ({ definitions, values, onChange }: Props) => {
  return (
    <div className="grid grid-cols-2 gap-4 border-t pt-4">
      {definitions.map((def) => (
        <div key={def.key} className="flex flex-col gap-1">
          <Label>{def.label}</Label>
          {def.type === "number" ? (
            <Input 
              type="number" 
              value={values[def.key] || ""} 
              onChange={(e) => onChange(def.key, e.target.value)} 
            />
          ) : (
            <Input 
              value={values[def.key] || ""} 
              onChange={(e) => onChange(def.key, e.target.value)} 
            />
          )}
        </div>
      ))}
    </div>
  );
};
```

## 🏁 Definition of Done (DoD)
- [ ] 모든 Acceptance Criteria를 충족하는가?
- [ ] 정의되지 않은 필드명이 JSONB에 무분별하게 쌓이지 않도록 스키마 검증(Validaton)이 동작하는가?
- [ ] UDF 정의가 수정/삭제될 때 기존 데이터와의 정합성 정책이 수립되었는가?

## 🚧 Dependencies & Blockers
- Depends on: `VIEW-010`, `DB-004`
- Blocks: 없음
