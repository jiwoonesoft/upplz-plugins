---
name: upplz-metadata
description: 앱 스토어 메타데이터(설명·키워드·릴리스 노트·스크린샷 등)를 조회(pull)하거나 반영(apply)할 때 사용. "메타데이터 수정", "설명 바꿔줘", "릴리스 노트 업데이트" 요청 시 활성화.
---

## Overview
스토어 메타데이터를 pull(현재 값 가져오기)하고 apply(변경 반영)하는 스킬. 빌드와 독립 — 코드 변경 없이 메타만 갱신 가능.

## When to Use
- 앱 설명/키워드/릴리스 노트/스크린샷 등 스토어 리스팅만 바꿀 때.

## Routing (첫 단계, 필수)
`${CLAUDE_PLUGIN_ROOT}/references/routing.md` 로드해 대상(iOS/Android)을 확정한다. iOS는 로컬 fastlane deliver(맥 필요, 비맥이면 베타2 게이트). Android는 `pull_android_metadata`/`apply_android_metadata` 정식 도구로 로컬 Docker(`web-game-android`, fastlane supply)를 통해 처리한다(빌드 하네스가 이미 존재하므로 iOS와 동일한 pull→편집→apply 흐름).

## Process
### iOS
1. `pull_ios_metadata` — 현재 스토어 메타데이터를 로컬로 가져온다(`ios/fastlane/metadata/**`).
2. 사용자가 로컬 파일 편집(설명/키워드/릴리스 노트/스크린샷 배치).
3. `apply_ios_metadata` — 변경 반영. scope 선택: 텍스트만=listing / 스크린샷만=screenshots / 둘 다=all.

### Android
1. `pull_android_metadata({ packageName })` — 현재 Play Store 리스팅을 로컬로 가져온다(`android/fastlane/metadata/android/**`). 응답의 Docker 명령(텍스트)과 Node 스크립트(이미지)를 사용자 맥에서 실행.
2. 사용자가 로컬 파일 편집(제목/설명/릴리스 노트/이미지·스크린샷 배치).
3. `apply_android_metadata({ packageName, scope })` — 변경 반영. scope 선택: 텍스트만=listing / 릴리스노트만=changelog / 이미지만=images / 전체=all. 응답의 Docker 명령을 사용자 맥에서 실행(`fastlane supply`, `web-game-android` 이미지 필요).
> Play 정책상 최초 업로드(앱 생성)가 아직이면 트랙에 릴리스가 없다는 에러가 날 수 있다 — `${CLAUDE_PLUGIN_ROOT}/references/first-time-setup.md`의 Android 절차(최초 수동 업로드)가 먼저 완료돼야 한다.

## Verification
- iOS: apply 성공, App Store Connect에 변경 반영.
- Android: apply 성공, Play Console 리스팅에 변경 반영.

## Next
`${CLAUDE_PLUGIN_ROOT}/references/chaining.md`: 메타데이터는 빌드와 독립이므로 보통 여기서 종료. 코드 변경도 있으면 upplz-upload-build-only 제안.
