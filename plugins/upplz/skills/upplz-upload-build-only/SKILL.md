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
`${CLAUDE_PLUGIN_ROOT}/references/routing.md`로 머신·대상 확정. iOS ∧ 비맥이면 iOS 맥 전용 게이트로 중단. Android 는 맥·윈도(Git Bash + Docker Desktop) 모두 같은 Process 다.

## 최초 셋업 게이트 (두 번째 단계, 필수)
`${CLAUDE_PLUGIN_ROOT}/references/first-time-setup.md`의 "셋업 완료 판정"으로 확인한다 — **iOS·Android 모두 `get_setup_status` 로 판정**하고(iOS 는 파라미터 없이, Android 는 `platform: "android"` — 패키지명을 알면 `packageName` 도 함께 넘겨 힌트의 owner 요청문에 실제 패키지명이 찍히게 한다. 선택이며 판정은 바뀌지 않는다), `setupComplete`면 Process로, 아니면 `nextStep`으로 분기한다(두 플랫폼이 같은 모양의 루프다). 미완료면 해당 플랫폼의 최초 셋업 절차를 수행한 뒤 아래 Process로 복귀한다(사용자가 `/upplz:init`을 아직 실행하지 않았다면 그 커맨드를 권하되, 요청하면 같은 절차를 여기서 그대로 수행한다 — 둘은 같은 문서를 쓴다). 진행 중 PRECONDITION/VERSION 계열 에러 코드가 오면 도구가 방어하는 것이므로, 지시대로 같은 문서의 해당 단계로 돌아간다(`get_setup_status` 를 같은 `platform` 으로 재호출해 복귀 지점을 잡는다).

## Process
대상=iOS면 1–5(iOS)를, 대상=Android면 1'–5'(Android)를 따른다. **역할로 갈리지 않는다** — 협업자도 빌드부터 업로드·보고까지 전부 한다.

**스크립트 실행 규칙(공통)**: 모든 스크립트 도구의 응답 `env` 는 프로젝트 비밀번호 `MATCH_PASSWORD` 하나다. 그 값을 export 한 뒤 응답의 스크립트를 **통째로** 실행하고, 끝나면 unset 한다. 스크립트가 인증서 저장소에서 필요한 파일(`.p8`·SA·keystore)을 풀어 쓰고 종료 시 지운다 — 키 파일을 직접 준비하거나 `Read` 하지 않는다. `[fetch] … 가 없습니다` 면 owner 가 지목된 등록 도구를 실행해야 한다. `[decrypt]` 면 먼저 응답 `env.MATCH_PASSWORD` 를 그대로 export 했는지 확인하고, 맞으면 응답 `troubleshooting.passwordMismatch` 를 그대로 전달한다(owner 가 그 파일의 등록 도구를 다시 실행 — 같은 저장소를 다른 upplz 프로젝트도 쓰면 대시보드 '다른 프로젝트에서 복사' 로 비밀번호를 맞춘다).

**하네스 우회 금지(공통)**: 빌드는 반드시 도구가 반환한 preflightScript/buildScript(iOS 로컬 스크립트, Android Docker 하네스)로만 수행한다. 레포 구조 불일치·preflight 실패·버전 주입 반영 확인 실패·UNSUPPORTED_PROJECT_TYPE 등으로 스크립트가 실패하면, **하네스를 우회한 임의 빌드(예: 로컬 gradle/xcodebuild 직접 실행)로 대체하지 않는다.** 중단하고 실패 원인과 해결책(구조 조정 방법 또는 미지원 사실)을 사용자에게 보고하며, 하네스 밖에서 만든 산출물을 upload_to_store/report_build_result로 이어가지 않는다.

### iOS
1. (수동) `git status` / `git log origin/main..HEAD`로 변경·미push 확인. **push 안 하면 이전 코드로 빌드된다.** 미push 커밋이 있으면 push.
2. 메타데이터(`ios/fastlane/metadata|screenshots/**`)만 바뀌었다면 → `apply_ios_metadata`만 하고 종료(빌드 불필요). upplz-upload에서 위임된 경우에는 종료하지 않고 upplz-upload의 다음 단계로 복귀한다.
3. `start_build` — `repoUrl` 전달(필수). projectType은 직전 빌드 값이 재사용된다(**최초 빌드이거나 native-ios/flutter/react-native인데 불확실하면 `projectType` 명시** — 미지정 시 web-capacitor로 잘못 빌드됨. 빌드 대상이 모호하면 `scheme`/`xcodeProjectPath` 지정). 반환된 로컬 빌드 스크립트를 응답 `env.MATCH_PASSWORD` 와 함께 실행 → `.upplz/App.ipa` 생성. **응답에 Apple API 키(.p8)는 들어 있지 않다** — buildScript 의 match 가 `--readonly` 라 저장소의 인증서·프로파일만 받는다. 프로젝트 비밀번호가 없으면 `PRECONDITION_FAILED` 로 멈춘다(owner 가 `register_ios_credentials` 의 `matchPassword` 로 저장해야 한다).
   - marketingVersion 미입력 시 MARKETING_VERSION_REQUIRED로 되물음. App Store에 이미 출시/심사된 버전(예: 1.0)이면 **더 높은 버전**(1.0.1)을 지정(train closed 예방). 빌드 번호는 자동 +1.
   - match 가 프로파일을 찾지 못해 실패하면 이 Bundle ID 의 프로파일이 저장소에 없다는 뜻이다 → `add_provisioning_profile` 로 추가한 뒤 다시 빌드한다(활성 멤버 누구나).
4. `upload_to_store` — 로컬 pilot 업로드. 스크립트의 프롤로그가 인증서 저장소에서 팀 키(`.p8`)를 풀어 쓰므로 응답 `env.MATCH_PASSWORD` 만 export 하면 된다. VERSION_ALREADY_USED면 3으로 돌아가 marketingVersion 상향. 실패하면 응답의 `appleErrorGuidance` 로 원인을 안내한다(예: ASC 앱 레코드가 없음 → ASC 웹 My Apps 에서 앱을 만든 뒤 재시도).
5. `report_build_result` — 결과 보고(iOS이고 실패가 아니면 스토어 공개 시 서버가 iTunes Lookup으로 쇼케이스 자동 보강 — showcase 생략 가능. 플랫폼 판정은 항상 빌드 값 기준).

### Android (맥 · 윈도)
1'. (수동) `git status` / `git log origin/main..HEAD`로 변경·미push 확인. **push 안 하면 이전 코드로 빌드된다.** 미push 커밋이 있으면 push.
2'. 메타데이터(`android/fastlane/metadata/android/**`)만 바뀌었다면 → `apply_android_metadata`만 하고 종료(빌드 불필요). upplz-upload에서 위임된 경우에는 종료하지 않고 upplz-upload의 다음 단계로 복귀한다.
3'. `start_build({ platform: "android", repoUrl, packageName, marketingVersion })` — `packageName`(미지정 시 `bundleId` 값으로 대체) 필수. projectType은 직전 android 빌드 값이 재사용된다(web-capacitor|native-android만 지원, 그 외 UNSUPPORTED_PROJECT_TYPE — 최초 빌드면 명시). preflightScript(Docker·이미지·openssl·**인증서 저장소 접근**·비밀번호 점검) + buildScript를 응답 `env.MATCH_PASSWORD` 와 함께 실행 → `.upplz/app-release.aab` 생성. **keystore 비밀번호는 프로젝트 비밀번호와 같은 값**이라 따로 받거나 정하지 않는다. PACKAGE_NAME_REQUIRED/MARKETING_VERSION_REQUIRED/BUILD_IN_PROGRESS 게이트는 도구가 방어.
   - **keystore 는 인증서 저장소(match 저장소)의 `android/<packageName>.keystore` 에서 받는다** — buildScript 가 사용자 본인 git 자격으로 shallow clone 해 서명하고 끝나면 임시 디렉토리를 지운다. 협업자도 같은 스크립트를 받으며, 그 저장소의 **GitHub collaborator** 여야 한다(접근 거부 시 owner 에게 추가 요청 — upplz 가 부여할 수 없다). 저장소에 keystore 가 없으면 스크립트가 "owner 가 `setup_android_keystore` 를 실행해야 한다"며 멈춘다 — 그 문구를 그대로 전달한다. 인증서 저장소 URL 이나 프로젝트 비밀번호 자체가 없으면 도구가 `PRECONDITION_FAILED` 로 막는다.
   - Apple Silicon 맥·윈도 모두 이미지는 `docker build --platform linux/amd64 -t web-game-android docker/android/` 로 만든 것이어야 한다(arm64 로 빌드된 옛 이미지는 pull 을 시도하며 실패한다).
   - marketingVersion 미입력 시 MARKETING_VERSION_REQUIRED로 되물음. Play Console에 이미 출시/심사된 버전이면 더 높은 버전을 지정. versionCode(빌드 번호)는 자동 증가.
   - gradle 위치(android/ 하위 또는 저장소 루트)와 Groovy/Kotlin DSL 버전 주입은 buildScript가 자동 처리한다 — 실패 시 위 "하네스 우회 금지" 규칙을 따른다.
4'. `upload_to_store({ platform: "android", buildId, packageName })` — 로컬 fastlane supply 업로드. 응답 `env.MATCH_PASSWORD` 를 export 한 뒤 스크립트를 **통째로** 실행하면, 프롤로그가 인증서 저장소에서 Play Service Account 를 임시 파일로 풀고 컨테이너에는 마운트 경로만 넘겼다가 종료 시 지운다. SA 미등록이면 `PRECONDITION_FAILED` — owner 에게 `register_android_credentials` 실행을 요청한다(owner 전용). **최초 배포는 이 단계 대신 Play Console 웹 수동 업로드가 필요** — `${CLAUDE_PLUGIN_ROOT}/references/first-time-setup.md`의 Android 8 참고. 실패 시 응답의 `playErrorGuidance`(예: PLAY_APP_NOT_FOUND → 최초 수동 업로드가 아직 안 됐다는 뜻)로 원인을 안내한다. PLATFORM_MISMATCH/PACKAGE_NAME_REQUIRED는 도구가 방어.
5'. `report_build_result({ buildId, status, showcase: { platform: "android", ... } })` — 결과 보고(아이콘이 있을 때만 수집, 대표 스크린샷은 선택. android는 자동 보강 미지원이므로 직접 제공).

## Verification
- **iOS**: `.upplz/App.ipa` 생성, 업로드 성공(App Store Connect 처리 중/완료), 보고 완료. Apple 처리(5–30분) 후 TestFlight에서 새 빌드 확인 가능함을 안내.
- **Android**: `.upplz/app-release.aab` 생성, 업로드 성공(최초 1회는 Play Console 웹 수동 업로드 완료), 보고 완료.

## Next
`${CLAUDE_PLUGIN_ROOT}/references/chaining.md`에 따라 물어본다.
- 출시까지 진행? → `upplz-upload`(메타데이터·아이콘·스크린샷 점검 포함).
- iOS: 테스터 추가?(add_testers — report_build_result로 uploaded 보고가 끝난 빌드만. 테스터 관리는 팀 키의 ASC 역할이 App Manager 이상이어야 한다. Android 에는 테스터 도구가 없다) / 아이콘·스크린샷·메타데이터 개별 갱신?(upplz-icon/upplz-screenshot/upplz-metadata).
- Android: 메타데이터 갱신?(upplz-metadata → apply_android_metadata). 아이콘/스크린샷 자동 생성·테스터 도구는 iOS 전용.
- 스토어 심사 제출은 수동(iOS: App Store Connect 웹 Add for Review → Submit for Review, Android: Play Console 웹 트랙별 검토·출시)임을 안내한다.
