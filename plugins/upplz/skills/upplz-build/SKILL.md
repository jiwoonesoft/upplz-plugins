---
name: upplz-build
description: 이미 셋업된 upplz 앱의 코드 변경 후 반복 빌드·업로드에 사용(iOS/Android). "다시 빌드", "새 버전 올려줘", "업데이트 배포" 요청 시 활성화. 최초 배포라면 upplz-onboard를 쓴다.
---

## Overview
셋업이 끝난 앱의 반복 빌드→업로드→보고 오케스트레이터(iOS/Android).

## When to Use
- iOS: Bundle ID·앱 레코드·프로비저닝·match가 이미 준비됨. Android: keystore·Play Console 앱(최초 AAB 수동 업로드)이 이미 준비됨.
- 코드 변경 후 새 빌드를 올릴 때.

## Routing (첫 단계, 필수)
`${CLAUDE_PLUGIN_ROOT}/references/routing.md`로 머신·대상 확정. iOS ∧ 비맥이면 베타2 게이트. Android ∧ 비맥(윈도)이면 정식 하네스 미제공(추후 지원) 안내.

## Process
대상=iOS면 1–5(iOS)를, 대상=Android(맥)면 1'–4'(Android)를 따른다.

### iOS
1. (수동) `git status` / `git log origin/main..HEAD`로 변경·미push 확인. **push 안 하면 이전 코드로 빌드된다.** 미push 커밋이 있으면 push.
2. 메타데이터(`ios/fastlane/metadata|screenshots/**`)만 바뀌었다면 → `apply_ios_metadata`만 하고 종료(빌드 불필요).
3. `start_build` — `repoUrl` 전달. projectType은 직전 빌드 값이 재사용된다(native 앱인데 직전 빌드가 없거나 불확실하면 `projectType` 명시). 반환된 로컬 빌드 스크립트 + 임시 api_key를 **`MATCH_PASSWORD` env**와 함께 실행 → `.upplz/App.ipa` 생성.
   - marketingVersion 미입력 시 MARKETING_VERSION_REQUIRED로 되물음. App Store에 이미 출시/심사된 버전(예: 1.0)이면 **더 높은 버전**(1.0.1)을 지정(train closed 예방). 빌드 번호는 자동 +1.
4. `upload_to_store` — 로컬 pilot 업로드. VERSION_ALREADY_USED면 3으로 돌아가 marketingVersion 상향.
5. `report_build_result` — 결과 보고.

### Android (베타1: 맥 전용)
1'. (수동) `git status` / `git log origin/main..HEAD`로 변경·미push 확인. **push 안 하면 이전 코드로 빌드된다.** 미push 커밋이 있으면 push.
2'. 메타데이터(`android/fastlane/metadata/android/**`)만 바뀌었다면 → `apply_android_metadata`만 하고 종료(빌드 불필요).
3'. `start_build({ platform: "android", repoUrl, packageName, marketingVersion })` — `packageName`(미지정 시 `bundleId` 값으로 대체) 필수. projectType은 직전 android 빌드 값이 재사용된다(web-capacitor|native-android만 지원, 그 외 UNSUPPORTED_PROJECT_TYPE). preflightScript(Docker·이미지·keystore·비밀번호 점검) + buildScript를 **`UPPLZ_KEYSTORE_PASSWORD` env**와 함께 사용자 맥에서 실행 → `.upplz/app-release.aab` 생성.
   - marketingVersion 미입력 시 MARKETING_VERSION_REQUIRED로 되물음. Play Console에 이미 출시/심사된 버전이면 더 높은 버전을 지정. versionCode(빌드 번호)는 자동 증가.
4'. `upload_to_store({ platform: "android", buildId, packageName })` — 로컬 fastlane supply 업로드(android/service-account.json, 최초 AAB가 아직 수동 업로드되지 않았다면 응답의 `playErrorGuidance.PLAY_APP_NOT_FOUND` 참고 → upplz-onboard의 Android 흐름 8번으로).
5'. `report_build_result({ buildId, status, showcase: { platform: "android", ... } })` — 결과 보고.

## Verification
- **iOS**: `.upplz/App.ipa` 생성, 업로드 성공, 보고 완료. Apple 처리(5–30분) 후 TestFlight에서 새 빌드 확인 가능함을 안내.
- **Android**: `.upplz/app-release.aab` 생성, 업로드 성공, 보고 완료.

## Next
`${CLAUDE_PLUGIN_ROOT}/references/chaining.md`에 따라 물어본다.
- iOS: 아이콘 갱신?(upplz-icon) / 스크린샷 갱신?(upplz-screenshot) / 메타데이터 갱신?(upplz-metadata) / 테스터 추가?(add_testers — report_build_result로 uploaded 보고가 끝난 빌드만).
- Android: 메타데이터 갱신?(upplz-metadata → apply_android_metadata). 아이콘/스크린샷 자동 생성·테스터 도구는 iOS 전용.
- 새 버전을 스토어에 출시하려면 심사 제출은 수동(iOS: App Store Connect 웹 Add for Review → Submit for Review, Android: Play Console 웹 트랙별 검토·출시)임을 안내한다.
