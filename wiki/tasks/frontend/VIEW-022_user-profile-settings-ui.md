---
name: Feature Task
about: SRS 기반의 구체적인 개발 태스크 명세
title: "[Feature] VIEW-022: 개인화 환경설정 및 사용자 프로필 관리 UI"
labels: 'feature, frontend, ux, priority:low'
assignees: ''
type: task
tags: [task, frontend]
created: 2026-04-22
---
## 🎯 Summary
- 기능명: [VIEW-022] 마이 프로필 및 사용자별 시스템 환경설정 화면
- 목적: B2B 유저 개개인의 생산성을 위해 아바타 변경, 선호 언어(KOR/ENG) 선택, 알림 수신 채널(알림톡/이메일) 토글 기능을 제공한다. 또한 다크모드/라이트모드 테마 설정을 브라우저 세션이 아닌 DB에 저장하여 로그인 시 언제든 복구되도록 한다.

## 🔗 References (Spec & Context)
> 💡 AI Agent & Dev Note: 작업 시작 전 아래 문서를 반드시 먼저 Read/Evaluate 할 것.
- 백엔드 의존성: `I18N-002` (Locale 파싱), `FILE-001` (이미지 업로드)
- UI 패턴: `VIEW-010` (설정 폼) 재활용

## ✅ Task Breakdown (실행 계획)
- [ ] `/settings/profile` 경로의 새로운 리액트 페이지 컴포넌트 개발
- [ ] 프로필 이미지 업로드 영역: 클릭 시 파일 탐색기 -> `FILE-001` API 호출 -> 즉시 프리뷰 업데이트
- [ ] `UserPreferenceForm` 구현:
  - 언어 선택 (ko/en) 드롭다운 (`I18N-001` 연계)
  - 테마 선택 (System/Light/Dark) 라디오 버튼
  - 알림 설정 (Daily Digest, 긴급 장애 알림) 체크박스
- [ ] 사용자 실명 및 연락처(PII) 수정 시 `SEC-004`(2FA) 인증 절차 결합
- [ ] `react-query` 를 사용하여 변경 사항 저장 시 전역 `AuthContext` 상태 동기화

## 🧪 Acceptance Criteria (BDD/GWT)
Scenario 1: 언어 설정 변경 시 즉시 반영
- Given: 현재 한국어 대시보드 상태
- When: 프로필 설정에서 'English' 선택 후 '저장' 클릭
- Then: 즉시 모든 메뉴 UI가 영문으로 번역되며, 이후 백엔드에서 전송되는 에러 메시지도 영문으로 수신된다.

Scenario 2: 프로필 이미지 업로드 및 유효성 검사
- Given: 프로필 편집 화면
- When: 5MB 크기의 `high_res.png` 파일을 업로드 시도함
- Then: 클라이언트 단에서 "파일 크기는 2MB를 초과할 수 없습니다" 경고를 띄우고 서버 요청을 차단한다.

## ⚙️ Technical & Non-Functional Constraints
- UX: '저장' 버튼 클릭 없이도 옵션 선택 즉시 로컬 반영(Auto-save) 로직 검토
- 가용성: 이미지 업로드 중 페이지 이탈 시 `FileUploadCancel` 처리
- 접근성: 폼 요소별 `Label` 및 `Focus` 상태 디자인 시스템 준수

## 💻 Implementation Snippet (Frontend Preference Logic)
```tsx
const ProfileSettings = () => {
    const { user, updateSettings } = useAuth();
    const { mutate: uploadAvatar } = useMutation(api.uploadProfileImage);

    const handleLanguageChange = (lng: string) => {
        i18n.changeLanguage(lng); // i18next 즉시 전환
        updateSettings({ language: lng }); // DB 영속화
    };

    return (
        <section className="max-w-2xl space-y-8">
            <AvatarUpload currentSrc={user.avatarUrl} onFileSelect={uploadAvatar} />
            <LanguageSelector current={i18n.language} onChange={handleLanguageChange} />
            <ThemeToggle current={user.theme} onChange={(t) => updateSettings({ theme: t })} />
        </section>
    );
};
```

## 🏁 Definition of Done (DoD)
- [ ] 모든 Acceptance Criteria를 충족하는가?
- [ ] 다국어 번역 리소스(`I18N-001`)에 설정 화면용 키값이 모두 등록되었는가?
- [ ] 브라우저를 새로고침해도 선택한 테마가 유지되는가?

## 🚧 Dependencies & Blockers
- Depends on: `FILE-001`, `I18N-001`, `AUTH-002`
- Blocks: 없음
