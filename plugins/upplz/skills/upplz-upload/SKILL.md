---
name: upplz-upload
description: 출시 목적으로 스토어에 올릴 때 사용. 아이콘·스크린샷·메타데이터를 모두 점검·갱신하면서 빌드→업로드→심사 제출 준비까지 진행한다(iOS/Android). "출시", "릴리스 준비", "스토어에 출시", "전체 배포", "종합 배포", "첫 출시" 요청 시 활성화. 테스트용 빌드 업로드만 필요하면 upplz-upload-build-only를 쓴다.
---

## Overview
출시 준비 풀 체인 오케스트레이터(iOS/Android). 아이콘·스크린샷·메타데이터를 순서대로 점검(상태 보고)하고 필요한 것만 갱신한 뒤, 빌드→업로드→보고를 거쳐 심사 제출(수동)을 안내한다. 최초/반복은 셋업 상태로 자동 분기한다.

## When to Use
- 스토어 **출시**(또는 출시 수준의 정비)가 목적일 때 — 메타·에셋까지 점검하고 배포를 끝내고 싶을 때.
- 빌드만 테스트 트랙에 올리면 되면 `upplz-upload-build-only`를 쓴다.

## Routing (첫 단계, 필수)
`${CLAUDE_PLUGIN_ROOT}/references/routing.md` 로드. iOS ∧ 비맥이면 베타2 게이트. Android ∧ 비맥(윈도)이면 정식 하네스 미제공(추후 지원) 안내.

## 최초 셋업 게이트 (두 번째 단계, 필수)
`${CLAUDE_PLUGIN_ROOT}/references/first-time-setup.md`의 "셋업 완료 판정"으로 확인. 미완료면 해당 플랫폼의 최초 셋업 절차를 수행한 뒤 아래 Process로 복귀한다.

## Process
출시 목적이므로 **점검(1–3) 자체는 건너뛰지 않고 각 항목의 상태를 확인·보고**하되, 갱신 실행 여부는 사용자에게 물어본다(원치 않으면 skip). Android는 아이콘·스크린샷 자동 생성 도구가 없으므로(iOS 전용) 1–2는 상태 확인·안내만 한다. Android의 확인 대상: 아이콘은 `android/app/src/main/res/mipmap-*`, 스크린샷·스토어 이미지는 3단계 `pull_android_metadata` 결과(`android/fastlane/metadata/android/**/images`)로 확인한다.

1. **아이콘 점검** — `AppIcon.appiconset`(1024 마스터) 존재·최신 여부 확인. 갱신 필요 시 `upplz-icon`(generate_app_icon) 진행. **아이콘 변경은 바이너리 재빌드 후 반영되므로 4 이전에 수행한다.**
2. **스크린샷 점검** — `ios/fastlane/screenshots/**` 존재·현재 앱 화면과 일치 여부 확인. 갱신 필요 시 `upplz-screenshot` 진행.
3. **메타데이터 점검** — iOS는 `pull_ios_metadata`, Android는 `pull_android_metadata({ packageName })`로 현재 스토어 값을 내려받아 설명·키워드·릴리스 노트를 확인. 갱신 필요 시 `upplz-metadata`(apply) 진행. 새 버전 출시라면 릴리스 노트 갱신을 권한다.
4. **빌드→업로드→보고** — `upplz-upload-build-only`의 Process(iOS 1–5 / Android 1'–5')를 그대로 따른다(`repoUrl`·`MATCH_PASSWORD` 또는 `packageName`·`UPPLZ_KEYSTORE_PASSWORD` env 필요). 아이콘/코드 변경이 있었다면 필수, 2–3만 갱신했고 코드·아이콘 변경이 없다면 빌드는 생략 가능(메타는 빌드와 독립).
5. (iOS 선택) 테스터 추가 → `add_testers`(선행: 4의 report_build_result로 uploaded 보고 완료).
6. **심사 제출 안내(수동)** — iOS: App Store Connect 웹 → 해당 버전 → Add for Review → Submit for Review. Android: Play Console 웹에서 트랙별 검토·출시. upplz 도구 범위 밖임을 안내한다.

## Verification
- 점검 항목(아이콘·스크린샷·메타데이터)별 상태가 사용자에게 보고됨. 갱신한 항목은 산출물 확인(appiconset/`ios/fastlane/screenshots/**`/apply 성공).
- 빌드 진행 시 iOS는 `.upplz/App.ipa`, Android는 `.upplz/app-release.aab` + 업로드 성공 + 보고 완료.

## Next
전체 완료. **스토어 심사 제출은 수동**임을 다시 안내한다(위 6). 이후 개별 작업이 필요하면 해당 스킬(upplz-icon/upplz-screenshot/upplz-metadata/upplz-upload-build-only)을 안내한다.
