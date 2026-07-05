# 최초 셋업 — iOS / Android (first-time setup)

`upplz-upload-build-only`·`upplz-upload` 스킬이 공용 참조하는 최초 배포 셋업 절차. 셋업 미완료로 판정되면 이 문서의 해당 플랫폼 절차를 수행한 뒤 **호출한 스킬의 Process(빌드 단계)로 복귀**한다.

## 셋업 완료 판정

- **iOS**: Apple API 크레덴셜(금고 등록)·match 레포·프로비저닝 프로파일이 준비됐는가. 불확실하면 그대로 빌드 단계로 진행해도 된다 — `start_build`가 PRECONDITION_FAILED/PROVISIONING_PROFILE_MISSING 등 에러 코드로 미비 단계를 알려주므로, 그 지시대로 이 문서의 해당 단계로 돌아온다.
- **Android**: 로컬 keystore 존재 + Play Console 앱 생성(최초 AAB 수동 업로드) 완료 여부. keystore가 없거나 Play Console에 앱이 없으면 아래 Android 절차.

## iOS 최초 셋업

1. `analyze_repository` — projectType·구조 확인.
2. 빌드 환경 확인 — 맥에 Xcode·Command Line Tools·ruby·fastlane 필요(미설치 시 설치 안내). 최종 점검은 `start_build` 응답의 preflightScript가 수행한다.
3. `setup_project` — 로컬 스캐폴드/설정(web-capacitor 전용, `iconUrl`로 초기 아이콘 지정 가능). native-ios/flutter/react-native는 건너뛴다.
4. (수동) 변경분 커밋·push — **push하지 않으면 start_build가 원격의 이전 코드로 빌드된다.**
5. `create_upload_link` — Apple API 크레덴셜(.p8/Key ID/Issuer ID/Team ID) **+ match 저장소 URL**(인증서 보관용 private repo) 등록. 사용자가 링크에서 입력.
6. `register_bundle_id` — 반환된 로컬 `fastlane produce` 스크립트를 사용자 맥에서 실행(Developer Portal Bundle ID 등록).
7. (수동) 사용자가 App Store Connect 웹에서 앱 레코드 생성.
8. `verify_app_record` — 앱 레코드 존재 확인. 없으면 7로.
   > (선택) 초기 메타데이터·스크린샷이 필요하면 이 시점부터 가능하다(앱 레코드 생성 후) — upplz-upload의 점검 단계 또는 upplz-metadata/upplz-screenshot 스킬 참조.
9. `setup_match_repo` — match 레포 초기화(로컬 스크립트). **실행 시 `MATCH_PASSWORD` env 필요 — 사용자가 정해 직접 보관한다(upplz 미저장, 분실 시 인증서 재발급).**
10. `add_provisioning_profile` — Bundle ID용 프로파일 발급(로컬 스크립트, `MATCH_PASSWORD` env). 익스텐션 있으면 각 bundleId마다.
11. → 호출한 스킬의 Process(iOS 빌드 단계)로 복귀. 첫 빌드 시 native-ios/flutter/react-native는 `projectType`을 반드시 명시한다(미지정 시 web-capacitor로 잘못 빌드됨. 빌드 대상이 모호하면 `scheme`/`xcodeProjectPath` 지정).

## Android 최초 셋업 (베타1: 맥 전용)

Android는 Apple 크레덴셜·임시 api_key 발급과 완전히 분리된 경로다. keystore·Google Service Account는 항상 사용자 로컬(맥 Docker `web-game-android` 컨테이너)에만 있고 서버로 전송/저장되지 않는다.

1. `analyze_repository` — repoType/framework/hasCapacitor/hasMobileSupport 확인(도구가 projectType을 직접 반환하지 않는다). 이 결과와 `android/` 디렉토리 존재 여부로 에이전트가 projectType(web-capacitor/native-android)을 판단한다.
2. Docker 이미지 준비 확인 — `web-game-android` 이미지가 없으면 `docker build --platform linux/amd64 -t web-game-android docker/android/`로 빌드하도록 안내(최종 점검은 `start_build` 응답의 preflightScript가 수행).
3. `setup_project({ platform: "android", bundleId: packageName, packageName, appName })` — `bundleId`는 zod 스키마상 항상 필수(빠뜨리면 도구 호출이 거부됨)이므로 Android에서도 반드시 전달해야 하며, 보통 `packageName`과 동일한 값을 넣는다. Capacitor Android 플랫폼 스캐폴드(web-capacitor 전용, `packageName` 미지정 시 `bundleId` 값을 패키지명으로 대체 — `packageName` 명시를 권장). native-android는 이미 `android/`가 있으므로 건너뛴다. **packageName에 언더스코어(`_`)가 포함되면 `bundleId` 필드 형식(하이픈만 허용)과 달라 그대로 넣을 수 없음 — 이 경우 `bundleId`에는 언더스코어를 뺀 유효 값을 넣고 `packageName`을 별도 지정한다.**
4. (수동) 변경분 커밋·push — **push하지 않으면 start_build가 원격의 이전 코드로 빌드된다.**
5. `setup_android_keystore({ packageName })` — keystore 생성 로컬 스크립트. **실행 전 `UPPLZ_KEYSTORE_PASSWORD`(8자 이상) env를 사용자가 직접 정해 export한다 — match 비밀번호(MATCH_PASSWORD)와 동일한 정책으로 upplz는 저장하지 않는다.** 이미 keystore가 있으면 재생성 없이 재사용(Play 서명 키는 교체 불가 자산).
6. (수동, 사전 안내) Google Play Developer 계정($25 일회성, 승인 최대 48시간) + Google Cloud Service Account JSON 발급 + Play Console에서 해당 SA에 API 권한 부여가 아직이면 안내한다. 또한 Play 정책상 **API 업로드는 최초 1회 Play Console 웹에서 AAB를 수동 업로드해 앱을 생성한 뒤부터만 가능**하다는 점을 미리 안내한다(실제 수행은 8).
7. → 호출한 스킬의 Process(Android 빌드 단계)로 복귀해 첫 빌드로 `.upplz/app-release.aab`를 생성한다.
8. (수동, 최초 1회만) **Play Console 웹 → 앱 만들기 → 생성된 `.upplz/app-release.aab`를 내부 테스트 트랙에 직접 업로드해 앱을 완성한다.** fastlane supply(API 업로드)는 앱이 이미 존재해야 동작하므로 자동화할 수 없다 — 최초 배포에서는 이 수동 업로드가 `upload_to_store`를 대체하며, `upload_to_store`는 반복 빌드부터 사용한다.
