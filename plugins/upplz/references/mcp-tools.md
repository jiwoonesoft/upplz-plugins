# MCP 원자 도구 계약 (요약)

스킬(오케스트레이터)이 호출하는 upplz MCP 도구. 상세 파라미터는 도구 자체 설명 참조. 사전조건 게이트는 도구가 최종 방어한다.

## 온보딩/셋업
- `analyze_repository` — 저장소 구조·projectType 분석.
- `create_upload_link` — Apple API 크레덴셜(.p8/Key ID/Issuer ID/Team ID) + match 저장소 URL 등록 링크 생성/저장(폼 5항목).
- `register_bundle_id` — 로컬 `fastlane produce` 스크립트 반환(Developer Portal Bundle ID 등록). ASC 앱은 사용자가 웹에서 생성.
- `verify_app_record` — ASC 앱 레코드 존재 확인(서버 읽기).
- `setup_match_repo` — match 레포 초기화 로컬 스크립트. 실행 시 MATCH_PASSWORD env 필요(사용자 보관, upplz 미저장).
- `add_provisioning_profile` — 프로비저닝 프로파일 발급 로컬 스크립트(bundleId별). 실행 시 MATCH_PASSWORD env 필요.
- `setup_project` — 로컬 스캐폴드(Capacitor 등) + 로컬 커밋. `platform`(`ios`(기본)|`android`) 파라미터로 대상 선택. `bundleId`는 항상 필수(zod 레벨). platform=ios(기본)는 `iconUrl` 파라미터로 초기 아이콘 지정 가능. platform=android는 `packageName`(미지정 시 `bundleId` 값을 패키지명으로 대체)으로 Capacitor Android 플랫폼을 추가/동기화한다(`android/` 디렉토리가 없어 `start_build`가 실패하는 경우 먼저 이 도구로 추가).
- `setup_android_keystore` — Android 서명용 keystore 생성 로컬 스크립트 반환(사용자 맥 Docker `web-game-android` 컨테이너에서 keytool 실행, 서버는 실행하지 않음). 파라미터 `packageName` 필수. 기존 keystore가 있으면 재생성 없이 재사용(Play 서명 키는 교체 불가 자산). 실행 시 `UPPLZ_KEYSTORE_PASSWORD` env(8자 이상) 필요 — **사용자가 직접 정해 보관, upplz는 저장하지 않음(match 비밀번호와 동일 정책)**.

## 빌드/업로드
- `start_build` — `platform`(`ios`(기본)|`android`) 파라미터. **iOS**: 로컬 빌드 스크립트 + 임시 api_key(.p8) + UPPLZ_* env 반환. `.upplz/App.ipa` 생성. 사전조건 미충족 시 PRECONDITION_FAILED / PROVISIONING_PROFILE_MISSING / MARKETING_VERSION_REQUIRED 게이트. repoUrl 필수. projectType 미지정 시 직전 빌드 값→web-capacitor 기본(native 첫 빌드는 반드시 명시, 모호 시 scheme/xcodeProjectPath). 스크립트 실행 시 MATCH_PASSWORD env 필요. **Android**: Apple 크레덴셜·PROVISIONING 게이트와 완전히 분리된 별도 경로(임시 api_key 미발급, Apple 크레덴셜 조회 없음) — preflightScript(Docker·이미지·keystore·비밀번호 점검) + buildScript(Docker+gradle bundleRelease) 반환, `.upplz/app-release.aab` 생성. `packageName`(미지정 시 `bundleId` 값으로 대체, 둘 다 없으면 PACKAGE_NAME_REQUIRED) 필수. `projectType`은 `web-capacitor`|`native-android`만 지원(그 외 UNSUPPORTED_PROJECT_TYPE). `marketingVersion` 미지정 시 MARKETING_VERSION_REQUIRED(Android도 동일 게이트). `BUILD_IN_PROGRESS`는 플랫폼 무관(같은 repoUrl 기준). 스크립트 실행 시 `UPPLZ_KEYSTORE_PASSWORD` env 필요(서버 미저장).
- `get_build_status` — 빌드 상태 조회.
- `upload_to_store` — `platform`(`ios`(기본)|`android`) 파라미터. **iOS**: 로컬 pilot 업로드 스크립트 반환. VERSION_ALREADY_USED 시 marketingVersion 상향 후 재빌드. **Android**: 크레덴셜 발급이 아닌 fastlane supply 스크립트만 반환(`android/service-account.json`은 로컬 파일로만 참조, 서버 미전송·미저장). buildId가 android 빌드가 아니면 PLATFORM_MISMATCH. 빌드 metadata에 packageName이 없고 파라미터도 없으면 PACKAGE_NAME_REQUIRED. `aabPath`(기본 `./.upplz/app-release.aab`)·`track`(기본 internal) 선택 파라미터. 응답의 `playErrorGuidance`(PLAY_APP_NOT_FOUND/SA_PERMISSION_DENIED/PACKAGE_NAME_MISMATCH/VERSION_CODE_ALREADY_USED)는 실행 도구가 직접 던지는 에러 코드가 아니라 fastlane 실행 결과 해석용 가이드 텍스트다 — **PLAY_APP_NOT_FOUND**: Play Console에 이 패키지명 앱이 아직 없다는 뜻이므로, Play 정책상 최초 1회는 Play Console 웹에서 AAB를 수동 업로드해 앱을 생성해야 이후 이 도구가 동작한다.
- `report_build_result` — 빌드/업로드 결과 DB 기록 + 쇼케이스. `showcase.platform`(`ios`(기본)|`android`)으로 대상 플랫폼을 지정한다(아이콘+대표 스크린샷 둘 다 있을 때만 수집).

## 메타데이터/에셋
- `pull_ios_metadata` / `apply_ios_metadata` — 스토어 메타데이터 pull/apply(로컬 fastlane deliver). scope=listing|screenshots|all.
- `pull_android_metadata` / `apply_android_metadata` — Android 스토어 리스팅(제목/설명/릴리스노트/아이콘/스크린샷/피처 그래픽) pull/apply(로컬 fastlane supply, `web-game-android` Docker 컨테이너). AAB 바이너리는 다루지 않는다. `packageName` 필수, `metadataPath`(기본 `android/fastlane/metadata/android`) 선택. `apply_android_metadata`의 `scope`(listing|changelog|images|all, 기본 all)로 범위 선택. 최초 AAB가 아직 수동 업로드되지 않은 앱은 `pull`/`apply` 모두 실패한다(troubleshooting 참고).
- `generate_ios_screenshots` — 웹앱(web-capacitor) 스크린샷 생성(Playwright) + frameit(프레임·제목).
- `generate_native_screenshots` — 네이티브 앱(native-ios/flutter/react-native) 스크린샷 생성(xcrun simctl 시뮬레이터, 사용자가 화면 이동 후 Enter로 캡처).
- `generate_app_icon` — 1024 마스터 아이콘 생성 로컬 스크립트(파생 사이즈는 Xcode가 빌드 시 자동 생성). `iconSource`는 이미지(URL/PNG) 또는 **SVG**(rsvg-convert 우선→magick 래스터화, ImageMagick은 항상 필요) — 소스 제공 또는 Claude가 설계한 SVG 생성 택일. optional `appiconsetPath` — 자동 감지(`find .`)가 실패할 때 AppIcon.appiconset 경로를 직접 지정. 크레덴셜 무관, 변경 후 재빌드 필요.

## 테스터
- `add_testers` — 로컬 `fastlane pilot` 테스터 추가 스크립트 + 임시 .p8. 그룹 hasAccessToAllBuilds=true면 기존 테스터에게 새 빌드 자동 노출.
