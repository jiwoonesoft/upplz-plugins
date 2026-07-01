---
name: upplz-metadata
description: 앱 스토어 메타데이터(설명·키워드·릴리스 노트·스크린샷 등)를 조회(pull)하거나 반영(apply)할 때 사용. "메타데이터 수정", "설명 바꿔줘", "릴리스 노트 업데이트" 요청 시 활성화.
---

## Overview
스토어 메타데이터를 pull(현재 값 가져오기)하고 apply(변경 반영)하는 스킬. 빌드와 독립 — 코드 변경 없이 메타만 갱신 가능.

## When to Use
- 앱 설명/키워드/릴리스 노트/스크린샷 등 스토어 리스팅만 바꿀 때.

## Routing (첫 단계, 필수)
`${CLAUDE_PLUGIN_ROOT}/references/routing.md` 로드. iOS는 로컬 fastlane deliver(맥 필요, 비맥이면 베타2 게이트). Android는 얇게 안내.

## Process
1. `pull_ios_metadata` — 현재 스토어 메타데이터를 로컬로 가져온다(`ios/fastlane/metadata/**`).
2. 사용자가 로컬 파일 편집(설명/키워드/릴리스 노트/스크린샷 배치).
3. `apply_ios_metadata` — 변경 반영. scope 선택: 텍스트만=listing / 스크린샷만=screenshots / 둘 다=all.

## Verification
- apply 성공, App Store Connect에 변경 반영.

## Next
`${CLAUDE_PLUGIN_ROOT}/references/chaining.md`: 메타데이터는 빌드와 독립이므로 보통 여기서 종료. 코드 변경도 있으면 upplz-build 제안.
