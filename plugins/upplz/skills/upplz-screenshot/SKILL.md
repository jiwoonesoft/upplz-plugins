---
name: upplz-screenshot
description: 앱 스토어용 스크린샷을 생성하고 디바이스 프레임·제목을 입힐 때 사용. "스크린샷 만들어줘", "스샷 갱신", "스토어 이미지 생성" 요청 시 활성화.
---

## Overview
스크린샷 생성 + frameit(디바이스 베젤·캡션 합성) 오케스트레이터.

## When to Use
- 스토어 스크린샷을 새로 만들거나 갱신할 때.

## Routing (첫 단계, 필수)
`${CLAUDE_PLUGIN_ROOT}/references/routing.md` 로드. iOS 로컬 fastlane(맥 필요, 비맥이면 베타2 게이트).

## Process
1. `generate_ios_screenshots` — 스크린샷 생성(필요 시 frameit로 프레임·제목 합성). 배경은 로컬 ImageMagick, 제목 폰트는 맥 시스템 한글 폰트.
2. (선택) `apply_ios_metadata` scope=screenshots — 생성된 스크린샷을 스토어에 반영.

## Verification
- 스크린샷 파일 생성(`ios/fastlane/screenshots/**`). apply 시 스토어 반영.

## Next
`${CLAUDE_PLUGIN_ROOT}/references/chaining.md`: 메타데이터 반영?(apply, upplz-metadata) / 재빌드?(upplz-build).
