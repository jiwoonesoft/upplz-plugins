---
name: upplz-upload-build-only
description: 테스트 목적으로 빌드만 만들어 스토어에 업로드할 때 사용(iOS TestFlight / Android 내부 트랙). 메타데이터·아이콘·스크린샷은 다루지 않는다. 최초 배포면 셋업(Apple/keystore)부터 자동 진행. "테스트 빌드", "TestFlight에 올려줘", "빌드만 올려줘", "다시 빌드", "새 버전 올려줘", "업데이트 배포", "처음 배포", "앱 등록", "온보딩" 요청 시 활성화. 출시 목적(메타·에셋 점검 포함)이면 upplz-upload를 쓴다.
---

## Overview
테스트용 순수 빌드→업로드→보고 오케스트레이터(iOS/Android). 최초/반복은 셋업 상태로 자동 분기한다 — 셋업 미완료면 최초 셋업 레퍼런스를 먼저 수행한다. 실제 작업은 MCP 원자 도구가 수행한다.

## When to Use
- 코드 변경 후 새 빌드를 테스트 트랙(TestFlight/내부 테스트)에 올릴 때.
- 이 저장소/앱을 upplz로 **처음** 배포할 때(셋업부터 첫 테스트 업로드까지).
- 메타데이터·아이콘·스크린샷 점검까지 포함한 **출시**가 목적이면 `upplz-upload`를 쓴다.

## Routing (첫 단계, 필수)
`${CLAUDE_PLUGIN_ROOT}/references/routing.md`로 머신·대상 확정. iOS ∧ 비맥이면 베타2 게이트. Android ∧ 비맥(윈도)이면 정식 하네스 미제공(추후 지원) 안내.

## 최초 셋업 게이트 (두 번째 단계, 필수)
`${CLAUDE_PLUGIN_ROOT}/references/first-time-setup.md`의 "셋업 완료 판정"으로 확인한다. 미완료면 해당 플랫폼의 최초 셋업 절차를 수행한 뒤 아래 Process로 복귀한다. 진행 중 PRECONDITION/PROVISIONING/VERSION 계열 에러 코드가 오면 도구가 방어하는 것이므로, 지시대로 같은 문서의 해당 단계로 돌아간다.

## Process
대상=iOS면 1–5(iOS)를, 대상=Android(맥)면 1'–5'(Android)를 따른다.

### iOS
1. (수동) `git status` / `git log origin/main..HEAD`로 변경·미push 확인. **push 안 하면 이전 코드로 빌드된다.** 미push 커밋이 있으면 push.
2. 메타데이터(`ios/fastlane/metadata|screenshots/**`)만 바뀌었다면 → `apply_ios_metadata`만 하고 종료(빌드 불필요). upplz-upload에서 위임된 경우에는 종료하지 않고 upplz-upload의 다음 단계로 복귀한다.
3. `start_build` — `repoUrl` 전달(필수). projectType은 직전 빌드 값이 재사용된다(**최초 빌드이거나 native-ios/flutter/react-native인데 불확실하면 `projectType` 명시** — 미지정 시 web-capacitor로 잘못 빌드됨. 빌드 대상이 모호하면 `scheme`/`xcodeProjectPath` 지정). 반환된 로컬 빌드 스크립트 + 임시 api_key를 **`MATCH_PASSWORD` env**와 함께 실행 → `.upplz/App.ipa` 생성.
   - marketingVersion 미입력 시 MARKETING_VERSION_REQUIRED로 되물음. App Store에 이미 출시/심사된 버전(예: 1.0)이면 **더 높은 버전**(1.0.1)을 지정(train closed 예방). 빌드 번호는 자동 +1.
4. `upload_to_store` — 로컬 pilot 업로드. VERSION_ALREADY_USED면 3으로 돌아가 marketingVersion 상향.
5. `report_build_result` — 결과 보고(iOS이고 실패가 아니면 스토어 공개 시 서버가 iTunes Lookup으로 쇼케이스 자동 보강 — showcase 생략 가능. 플랫폼 판정은 항상 빌드 값 기준).

### Android (베타1: 맥 전용)
1'. (수동) `git status` / `git log origin/main..HEAD`로 변경·미push 확인. **push 안 하면 이전 코드로 빌드된다.** 미push 커밋이 있으면 push.
2'. 메타데이터(`android/fastlane/metadata/android/**`)만 바뀌었다면 → `apply_android_metadata`만 하고 종료(빌드 불필요). upplz-upload에서 위임된 경우에는 종료하지 않고 upplz-upload의 다음 단계로 복귀한다.
3'. `start_build({ platform: "android", repoUrl, packageName, marketingVersion })` — `packageName`(미지정 시 `bundleId` 값으로 대체) 필수. projectType은 직전 android 빌드 값이 재사용된다(web-capacitor|native-android만 지원, 그 외 UNSUPPORTED_PROJECT_TYPE — 최초 빌드면 명시). preflightScript(Docker·이미지·keystore·비밀번호 점검) + buildScript를 **`UPPLZ_KEYSTORE_PASSWORD` env**와 함께 사용자 맥에서 실행 → `.upplz/app-release.aab` 생성. PACKAGE_NAME_REQUIRED/MARKETING_VERSION_REQUIRED/BUILD_IN_PROGRESS 게이트는 도구가 방어.
   - marketingVersion 미입력 시 MARKETING_VERSION_REQUIRED로 되물음. Play Console에 이미 출시/심사된 버전이면 더 높은 버전을 지정. versionCode(빌드 번호)는 자동 증가.
4'. `upload_to_store({ platform: "android", buildId, packageName })` — 로컬 fastlane supply 업로드(android/service-account.json, 서버는 크레덴셜을 발급/보관하지 않음). **최초 배포는 이 단계 대신 Play Console 웹 수동 업로드가 필요** — `${CLAUDE_PLUGIN_ROOT}/references/first-time-setup.md`의 Android 8 참고. 실패 시 응답의 `playErrorGuidance`(예: PLAY_APP_NOT_FOUND → 최초 수동 업로드가 아직 안 됐다는 뜻)로 원인을 안내한다. PLATFORM_MISMATCH/PACKAGE_NAME_REQUIRED는 도구가 방어.
5'. `report_build_result({ buildId, status, showcase: { platform: "android", ... } })` — 결과 보고(아이콘이 있을 때만 수집, 대표 스크린샷은 선택. android는 자동 보강 미지원이므로 직접 제공).

## Verification
- **iOS**: `.upplz/App.ipa` 생성, 업로드 성공(App Store Connect 처리 중/완료), 보고 완료. Apple 처리(5–30분) 후 TestFlight에서 새 빌드 확인 가능함을 안내.
- **Android**: `.upplz/app-release.aab` 생성, 업로드 성공(최초 1회는 Play Console 웹 수동 업로드 완료), 보고 완료.

## Next
`${CLAUDE_PLUGIN_ROOT}/references/chaining.md`에 따라 물어본다.
- 출시까지 진행? → `upplz-upload`(메타데이터·아이콘·스크린샷 점검 포함).
- iOS: 테스터 추가?(add_testers — report_build_result로 uploaded 보고가 끝난 빌드만) / 아이콘·스크린샷·메타데이터 개별 갱신?(upplz-icon/upplz-screenshot/upplz-metadata).
- Android: 메타데이터 갱신?(upplz-metadata → apply_android_metadata). 아이콘/스크린샷 자동 생성·테스터 도구는 iOS 전용.
- 스토어 심사 제출은 수동(iOS: App Store Connect 웹 Add for Review → Submit for Review, Android: Play Console 웹 트랙별 검토·출시)임을 안내한다.
