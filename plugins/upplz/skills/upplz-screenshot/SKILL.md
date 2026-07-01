---
name: upplz-screenshot
description: 앱 스토어용 스크린샷을 생성하고 디바이스 프레임·제목을 입힐 때 사용. "스크린샷 만들어줘", "스샷 갱신", "스토어 이미지 생성" 요청 시 활성화.
---

## Overview
스토어 스크린샷 생성 오케스트레이터. **projectType(앱 유형)에 따라 캡처 방법을 택한다** — 웹앱은 Playwright, 네이티브는 시뮬레이터.

## When to Use
- 스토어 스크린샷을 새로 만들거나 갱신할 때.

## Routing (첫 단계, 필수)
`${CLAUDE_PLUGIN_ROOT}/references/routing.md` 로드 + **projectType 확인**. iOS 캡처는 맥 필요(비맥이면 베타2 게이트).

## Process
projectType으로 캡처 방법을 분기한다:

**web-capacitor (웹 게임) → `generate_ios_screenshots` (Playwright 웹 캡처)**
1. 웹앱을 로컬 dev 서버(예: `http://localhost:5173`)나 배포 URL로 띄운다.
2. `generate_ios_screenshots({ appUrl, locale, routes })` — 반환 스크립트로 캡처(필요 시 frameit로 프레임·제목 합성; 배경 로컬 ImageMagick, 제목 폰트 맥 시스템 한글 폰트).

**native-ios / flutter / react-native (네이티브) → `generate_native_screenshots` (xcrun simctl 시뮬레이터 캡처)**
1. **시뮬레이터용 빌드(.app)** 준비 — Xcode에서 시뮬레이터 대상 빌드(⌘B) 또는 `xcodebuild -sdk iphonesimulator … build`. upplz 정식 빌드의 기기용 `.ipa`는 스크린샷에 쓸 수 없다(시뮬레이터 `.app` 필요).
2. `generate_native_screenshots({ bundleId, locale, devices })` — 반환 스크립트를 사용자 맥에서 실행: 시뮬레이터 부팅 → 앱 설치·실행 → **가이드 캡처**(각 화면을 띄우고 Enter). 완전 자동 다중 화면은 fastlane snapshot(UITest 스킴 필요) 권장.

3. (선택) `apply_ios_metadata` scope=screenshots — 스토어 반영. 두 도구 모두 `ios/fastlane/screenshots/<locale>/<device>-<순번>.png` 레이아웃이라 apply 단계는 동일.

## Verification
- 스크린샷 파일 생성(`ios/fastlane/screenshots/**`). apply 시 스토어 반영.

## Next
`${CLAUDE_PLUGIN_ROOT}/references/chaining.md`: 메타데이터 반영?(apply, upplz-metadata) / 재빌드?(upplz-build).
