---
name: upplz-onboard
description: upplz로 앱을 처음 배포할 때 사용. Apple 셋업(Bundle ID·앱 레코드·서명)부터 첫 빌드·업로드까지 최초 온보딩 전 과정을 안내한다. "처음 배포", "앱 등록", "첫 출시", "온보딩" 요청 시 활성화.
---

## Overview
upplz iOS 최초 배포 온보딩. 환경 설치 → Apple 셋업 → 서명 준비 → 첫 빌드/업로드 → 결과 보고까지 오케스트레이션한다. 실제 작업은 MCP 원자 도구가 수행한다.

## When to Use
- 이 저장소/앱을 upplz로 **처음** 배포할 때.
- Apple Bundle ID·앱 레코드·프로비저닝이 아직 없을 때.
- 반복 빌드(이미 셋업 완료)라면 `upplz-build`를 쓴다.

## Routing (첫 단계, 필수)
`${CLAUDE_PLUGIN_ROOT}/references/routing.md`를 로드해 머신·대상을 확정한다. 대상=iOS ∧ 머신≠맥이면 베타2 게이트 안내 후 중단.

## Process
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

## Verification
- `.upplz/App.ipa` 생성됨.
- 업로드 성공(App Store Connect 처리 중/완료).
- `report_build_result` 호출 완료.
- Apple 처리(5–30분) 후 TestFlight 앱에서 빌드 확인 가능 — 처리 대기를 사용자에게 안내했는가.

## Next
- **스토어 심사 제출은 수동**: App Store Connect 웹 → 해당 버전 → Add for Review → Submit for Review. upplz 도구 범위 밖임을 안내한다.
- `${CLAUDE_PLUGIN_ROOT}/references/chaining.md`에 따라 사용자에게 물어본다: 스크린샷 생성?(upplz-screenshot) / 메타데이터 정리?(upplz-metadata) / 테스터 추가?(add_testers — report_build_result로 uploaded 보고가 끝난 빌드만 가능). 사용자가 원할 때만 이어서 진행한다.
