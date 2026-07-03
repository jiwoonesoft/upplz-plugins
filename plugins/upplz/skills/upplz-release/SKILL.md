---
name: upplz-release
description: 빌드부터 종합 처리까지 한 번에 진행할 때 사용(iOS/Android). 아이콘·스크린샷·메타데이터 갱신 → 빌드 → 업로드 → 테스터 → 보고를 각 단계 물어보며 순차 실행한다. "전체 배포", "한 번에 처리", "종합 배포" 요청 시 활성화.
---

## Overview
개별 스킬을 조합한 풀 체인 오케스트레이터(iOS/Android). 최초/반복은 사전조건(금고 시크릿·keystore 유무)으로 자동 분기한다.

## When to Use
- 코드·에셋·메타를 한꺼번에 갱신하고 배포까지 끝내고 싶을 때.

## Routing (첫 단계, 필수)
`${CLAUDE_PLUGIN_ROOT}/references/routing.md` 로드. iOS ∧ 비맥이면 베타2 게이트. Android ∧ 비맥(윈도)이면 정식 하네스 미제공(추후 지원) 안내.

## Process
각 단계를 **사용자에게 물어보며** 순차 진행(원치 않는 단계는 skip). 대상=Android(맥)면 3–4단계(아이콘/스크린샷 자동 생성·테스터)는 iOS 전용 도구이므로 건너뛴다.

### iOS
1. 최초 배포 여부 판단 — 셋업이 안 됐으면 `upplz-onboard` 흐름으로 위임 후 종료.
2. 아이콘 갱신? → `upplz-icon`(generate_app_icon).
3. 스크린샷 갱신? → `upplz-screenshot`.
4. 메타데이터 갱신? → `upplz-metadata`(apply_ios_metadata).
5. 빌드→업로드 → `upplz-build`(start_build → upload_to_store, `repoUrl`·`MATCH_PASSWORD` env 필요). 아이콘/코드 변경이 있었다면 필수.
6. `report_build_result` — 결과 보고.
7. 테스터 추가? → `add_testers`(선행: 6의 uploaded 보고 완료).

### Android (베타1: 맥 전용)
1. 최초 배포 여부 판단 — keystore·Play Console 앱(최초 AAB 수동 업로드)이 없으면 `upplz-onboard`의 Android 흐름으로 위임 후 종료.
2. 메타데이터 갱신? → `upplz-metadata`(apply_android_metadata, 선행: `pull_android_metadata`로 기존 값 확인 가능).
3. 빌드→업로드 → `upplz-build`의 Android 흐름(`start_build({ platform: "android" })` → `upload_to_store({ platform: "android" })`, `packageName`·`UPPLZ_KEYSTORE_PASSWORD` env 필요). 코드 변경이 있었다면 필수.
4. `report_build_result({ showcase: { platform: "android", ... } })` — 결과 보고.

## Verification
- 선택한 단계별 산출물 확인. 빌드 진행 시 iOS는 `.upplz/App.ipa`, Android는 `.upplz/app-release.aab` + 업로드 성공 + 보고 완료.

## Next
전체 완료. **스토어 심사 제출은 수동**: iOS는 App Store Connect 웹(Add for Review → Submit for Review), Android는 Play Console 웹(트랙별 검토·출시)에서 진행함을 안내한다. 후속 개별 작업이 필요하면 해당 스킬을 안내한다.
