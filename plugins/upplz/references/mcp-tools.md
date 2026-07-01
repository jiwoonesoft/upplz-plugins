# MCP 원자 도구 계약 (요약)

스킬(오케스트레이터)이 호출하는 upplz MCP 도구. 상세 파라미터는 도구 자체 설명 참조. 사전조건 게이트는 도구가 최종 방어한다.

## 온보딩/셋업
- `analyze_repository` — 저장소 구조·projectType 분석.
- `create_upload_link` — Apple API 크레덴셜(.p8/Key/Issuer/Team) 등록 링크 생성/저장.
- `register_bundle_id` — 로컬 `fastlane produce` 스크립트 반환(Developer Portal Bundle ID 등록). ASC 앱은 사용자가 웹에서 생성.
- `verify_app_record` — ASC 앱 레코드 존재 확인(서버 읽기).
- `setup_match_repo` — match 레포 초기화 로컬 스크립트.
- `add_provisioning_profile` — 프로비저닝 프로파일 발급 로컬 스크립트(bundleId별).
- `setup_project` — 로컬 스캐폴드(Capacitor 등) + 로컬 커밋. `iconUrl` 파라미터 지원.

## 빌드/업로드
- `start_build` — 로컬 빌드 스크립트 + 임시 api_key(.p8) + UPPLZ_* env 반환. `.upplz/App.ipa` 생성. 사전조건 미충족 시 PRECONDITION_FAILED / PROVISIONING_PROFILE_MISSING / MARKETING_VERSION_REQUIRED 게이트.
- `get_build_status` — 빌드 상태 조회.
- `upload_to_store` — 로컬 pilot 업로드 스크립트 반환. VERSION_ALREADY_USED 시 marketingVersion 상향 후 재빌드.
- `report_build_result` — 빌드/업로드 결과 DB 기록 + 쇼케이스.

## 메타데이터/에셋
- `pull_ios_metadata` / `apply_ios_metadata` — 스토어 메타데이터 pull/apply(로컬 fastlane deliver). scope=listing|screenshots|all.
- `pull_android_metadata` / `apply_android_metadata` — Android 메타데이터(얇게).
- `generate_ios_screenshots` — 웹앱(web-capacitor) 스크린샷 생성(Playwright) + frameit(프레임·제목).
- `generate_native_screenshots` — 네이티브 앱(native-ios/flutter/react-native) 스크린샷 생성(xcrun simctl 시뮬레이터, 사용자가 화면 이동 후 Enter로 캡처).
- `generate_app_icon` — 1024 마스터 아이콘 생성 로컬 스크립트(파생 사이즈는 Xcode가 빌드 시 자동 생성). `iconSource`는 이미지(URL/PNG) 또는 **SVG**(rsvg-convert→magick로 래스터화) — 소스 제공 또는 Claude가 설계한 SVG 생성 택일. optional `appiconsetPath` — 자동 감지(`find .`)가 실패할 때 AppIcon.appiconset 경로를 직접 지정. 크레덴셜 무관, 변경 후 재빌드 필요.

## 테스터
- `add_testers` — 로컬 `fastlane pilot` 테스터 추가 스크립트 + 임시 .p8. 그룹 hasAccessToAllBuilds=true면 기존 테스터에게 새 빌드 자동 노출.
