---
name: upplz-onboard
description: upplz로 앱을 처음 배포할 때 사용. iOS는 Apple 셋업(Bundle ID·앱 레코드·서명), Android는 keystore·Play Console 셋업부터 첫 빌드·업로드까지 최초 온보딩 전 과정을 안내한다. "처음 배포", "앱 등록", "첫 출시", "온보딩" 요청 시 활성화.
---

## Overview
upplz 최초 배포 온보딩(iOS/Android). 환경 설치 → 플랫폼별 셋업(Apple 서명 또는 Android keystore/Play Console) → 첫 빌드/업로드 → 결과 보고까지 오케스트레이션한다. 실제 작업은 MCP 원자 도구가 수행한다.

## When to Use
- 이 저장소/앱을 upplz로 **처음** 배포할 때.
- iOS: Apple Bundle ID·앱 레코드·프로비저닝이 아직 없을 때. Android: keystore·Play Console 앱이 아직 없을 때.
- 반복 빌드(이미 셋업 완료)라면 `upplz-build`를 쓴다.

## Routing (첫 단계, 필수)
`${CLAUDE_PLUGIN_ROOT}/references/routing.md`를 로드해 머신·대상을 확정한다. 대상=iOS ∧ 머신≠맥이면 베타2 게이트 안내 후 중단. 대상=Android ∧ 머신=맥이면 아래 "Process — Android"를, 대상=Android ∧ 머신≠맥(윈도)이면 정식 하네스 미제공(추후 지원) 안내 후 얇게만 돕는다.

## Process — iOS
1. `analyze_repository` — projectType·구조 확인.
2. 빌드 환경 확인 — 맥에 Xcode·Command Line Tools·ruby·fastlane 필요(미설치 시 설치 안내). 최종 점검은 `start_build` 응답의 preflightScript가 수행한다.
3. `setup_project` — 로컬 스캐폴드/설정(web-capacitor 전용, `iconUrl`로 초기 아이콘 지정 가능). native-ios/flutter/react-native는 건너뛴다.
4. (수동) 변경분 커밋·push — **push하지 않으면 start_build가 원격의 이전 코드로 빌드된다.**
5. `create_upload_link` — Apple API 크레덴셜(.p8/Key ID/Issuer ID/Team ID) **+ match 저장소 URL**(인증서 보관용 private repo) 등록. 사용자가 링크에서 입력.
6. `register_bundle_id` — 반환된 로컬 `fastlane produce` 스크립트를 사용자 맥에서 실행(Developer Portal Bundle ID 등록).
7. (수동) 사용자가 App Store Connect 웹에서 앱 레코드 생성.
8. `verify_app_record` — 앱 레코드 존재 확인. 없으면 7로.
9. (선택) 초기 메타데이터 — 웹 게임이면 `generate_ios_screenshots`, 네이티브 앱이면 `generate_native_screenshots` / `apply_ios_metadata`.
10. `setup_match_repo` — match 레포 초기화(로컬 스크립트). **실행 시 `MATCH_PASSWORD` env 필요 — 사용자가 정해 직접 보관한다(upplz 미저장, 분실 시 인증서 재발급).**
11. `add_provisioning_profile` — Bundle ID용 프로파일 발급(로컬 스크립트, `MATCH_PASSWORD` env). 익스텐션 있으면 각 bundleId마다.
12. `start_build` — **`repoUrl` 전달(필수), native-ios/flutter/react-native는 `projectType`도 반드시 명시**(미지정 시 web-capacitor로 잘못 빌드됨. 빌드 대상이 모호하면 `scheme`/`xcodeProjectPath` 지정). 반환된 로컬 빌드 스크립트 + 임시 api_key를 `MATCH_PASSWORD` env와 함께 실행 → `.upplz/App.ipa` 생성. MARKETING_VERSION_REQUIRED 시 사용자에게 버전 확인.
13. `upload_to_store` — 로컬 pilot 업로드.
14. `report_build_result` — 결과 보고 + 쇼케이스.
사전조건 게이트(PRECONDITION/PROVISIONING/VERSION)는 도구가 방어하므로, 에러 코드가 오면 지시대로 앞 단계로 돌아간다.

## Process — Android (베타1: 맥 전용)
Android는 Apple 크레덴셜·임시 api_key 발급과 완전히 분리된 경로다. keystore·Google Service Account는 항상 사용자 로컬(맥 Docker `web-game-android` 컨테이너)에만 있고 서버로 전송/저장되지 않는다.
1. `analyze_repository` — repoType/framework/hasCapacitor/hasMobileSupport 확인(도구가 projectType을 직접 반환하지 않는다). 이 결과와 `android/` 디렉토리 존재 여부로 에이전트가 projectType(web-capacitor/native-android)을 판단한다.
2. Docker 이미지 준비 확인 — `web-game-android` 이미지가 없으면 `docker build --platform linux/amd64 -t web-game-android docker/android/`로 빌드하도록 안내(최종 점검은 `start_build` 응답의 preflightScript가 수행).
3. `setup_project({ platform: "android", bundleId: packageName, packageName, appName })` — `bundleId`는 zod 스키마상 항상 필수(빠뜨리면 도구 호출이 거부됨)이므로 Android에서도 반드시 전달해야 하며, 보통 `packageName`과 동일한 값을 넣는다. Capacitor Android 플랫폼 스캐폴드(web-capacitor 전용, `packageName` 미지정 시 `bundleId` 값을 패키지명으로 대체 — `packageName` 명시를 권장). native-android는 이미 `android/`가 있으므로 건너뛴다. **packageName에 언더스코어(`_`)가 포함되면 `bundleId` 필드 형식(하이픈만 허용)과 달라 그대로 넣을 수 없음 — 이 경우 `bundleId`에는 언더스코어를 뺀 유효 값을 넣고 `packageName`을 별도 지정한다.**
4. (수동) 변경분 커밋·push — **push하지 않으면 start_build가 원격의 이전 코드로 빌드된다.**
5. `setup_android_keystore({ packageName })` — keystore 생성 로컬 스크립트. **실행 전 `UPPLZ_KEYSTORE_PASSWORD`(8자 이상) env를 사용자가 직접 정해 export한다 — match 비밀번호(MATCH_PASSWORD)와 동일한 정책으로 upplz는 저장하지 않는다.** 이미 keystore가 있으면 재생성 없이 재사용(Play 서명 키는 교체 불가 자산).
6. (수동, 사전 안내) Google Play Developer 계정($25 일회성, 승인 최대 48시간) + Google Cloud Service Account JSON 발급 + Play Console에서 해당 SA에 API 권한 부여가 아직이면 안내한다. 또한 Play 정책상 **API 업로드는 최초 1회 Play Console 웹에서 AAB를 수동 업로드해 앱을 생성한 뒤부터만 가능**하다는 점을 미리 안내한다(실제 수행은 8번).
7. `start_build({ platform: "android", packageName, marketingVersion, projectType })` — preflightScript(Docker 데몬·이미지·keystore·비밀번호 점검) + buildScript(Docker+gradle bundleRelease) 반환. `UPPLZ_KEYSTORE_PASSWORD` env와 함께 사용자 맥에서 실행 → `.upplz/app-release.aab` 생성. PACKAGE_NAME_REQUIRED/MARKETING_VERSION_REQUIRED/UNSUPPORTED_PROJECT_TYPE/BUILD_IN_PROGRESS 게이트는 도구가 방어.
8. (수동, 최초 1회만) **Play Console 웹 → 앱 만들기 → 방금 생성된 `.upplz/app-release.aab`를 내부 테스트 트랙에 직접 업로드해 앱을 완성한다.** fastlane supply(API 업로드)는 앱이 이미 존재해야 동작하므로 자동화할 수 없다.
9. `upload_to_store({ platform: "android", buildId, packageName })` — **반복 빌드부터** 사용하는 자동 업로드. 사용자 로컬 `android/service-account.json`으로 fastlane supply 실행(서버는 크레덴셜을 발급/보관하지 않음). PLATFORM_MISMATCH/PACKAGE_NAME_REQUIRED는 도구가 방어. 실패 시 응답의 `playErrorGuidance`(예: PLAY_APP_NOT_FOUND → 8번이 아직 안 됐다는 뜻)로 원인을 안내한다.
10. `report_build_result({ buildId, status, showcase: { platform: "android", ... } })` — 결과 보고 + 쇼케이스(아이콘+대표 스크린샷 둘 다 있을 때만 수집).
11. (선택) `apply_android_metadata` — 스토어 리스팅(제목/설명/릴리스노트/아이콘/스크린샷) 적용. 기존 앱이면 먼저 `pull_android_metadata`로 현재 값을 내려받아 편집.

## Verification
- **iOS**: `.upplz/App.ipa` 생성됨. 업로드 성공(App Store Connect 처리 중/완료). `report_build_result` 호출 완료. Apple 처리(5–30분) 후 TestFlight 앱에서 빌드 확인 가능 — 처리 대기를 사용자에게 안내했는가.
- **Android**: `.upplz/app-release.aab` 생성됨. 최초 1회는 Play Console 웹 수동 업로드 완료, 이후 `upload_to_store`(fastlane supply) 성공. `report_build_result` 호출 완료.

## Next
- **스토어 심사 제출은 수동**: iOS는 App Store Connect 웹 → 해당 버전 → Add for Review → Submit for Review. Android는 Play Console 웹에서 트랙별로 검토·출시. upplz 도구 범위 밖임을 안내한다.
- `${CLAUDE_PLUGIN_ROOT}/references/chaining.md`에 따라 사용자에게 물어본다: iOS는 스크린샷 생성?(upplz-screenshot) / 메타데이터 정리?(upplz-metadata) / 테스터 추가?(add_testers — report_build_result로 uploaded 보고가 끝난 빌드만 가능). Android는 메타데이터 정리?(upplz-metadata → apply_android_metadata)만 해당(아이콘/스크린샷 자동 생성·테스터 도구는 iOS 전용). 사용자가 원할 때만 이어서 진행한다.
