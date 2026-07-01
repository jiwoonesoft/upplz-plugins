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
2. `create_upload_link` — Apple API 크레덴셜(.p8/Key/Issuer/Team) 등록. 사용자가 링크에서 입력.
3. `register_bundle_id` — 반환된 로컬 `fastlane produce` 스크립트를 사용자 맥에서 실행(Developer Portal Bundle ID 등록).
4. (수동) 사용자가 App Store Connect 웹에서 앱 레코드 생성.
5. `verify_app_record` — 앱 레코드 존재 확인. 없으면 4로.
6. (선택) 초기 메타데이터 — 웹 게임이면 `generate_ios_screenshots`, 네이티브 앱이면 `generate_native_screenshots` / `apply_ios_metadata`.
7. `setup_match_repo` — match 레포 초기화(로컬 스크립트).
8. `add_provisioning_profile` — Bundle ID용 프로파일 발급(로컬 스크립트). 익스텐션 있으면 각 bundleId마다.
9. `setup_project` — 로컬 스캐폴드/설정(필요 시).
10. `start_build` — 로컬 빌드 스크립트 + 임시 api_key 실행. `.upplz/App.ipa` 생성. MARKETING_VERSION_REQUIRED 시 사용자에게 버전 확인.
11. `upload_to_store` — 로컬 pilot 업로드.
12. `report_build_result` — 결과 보고 + 쇼케이스.
사전조건 게이트(PRECONDITION/PROVISIONING/VERSION)는 도구가 방어하므로, 에러 코드가 오면 지시대로 앞 단계로 돌아간다.

## Verification
- `.upplz/App.ipa` 생성됨.
- 업로드 성공(App Store Connect 처리 중/완료).
- `report_build_result` 호출 완료.

## Next
`${CLAUDE_PLUGIN_ROOT}/references/chaining.md`에 따라 사용자에게 물어본다: 스크린샷 생성?(upplz-screenshot) / 메타데이터 정리?(upplz-metadata) / 테스터 추가?(add_testers). 사용자가 원할 때만 이어서 진행한다.
