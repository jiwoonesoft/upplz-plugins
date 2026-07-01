---
name: upplz-build
description: 이미 셋업된 upplz 앱의 코드 변경 후 반복 빌드·업로드에 사용. "다시 빌드", "새 버전 올려줘", "업데이트 배포" 요청 시 활성화. 최초 배포라면 upplz-onboard를 쓴다.
---

## Overview
셋업이 끝난 iOS 앱의 반복 빌드→업로드→보고 오케스트레이터.

## When to Use
- Bundle ID·앱 레코드·프로비저닝·match가 이미 준비됨.
- 코드 변경 후 새 빌드를 올릴 때.

## Routing (첫 단계, 필수)
`${CLAUDE_PLUGIN_ROOT}/references/routing.md`로 머신·대상 확정. iOS ∧ 비맥이면 베타2 게이트.

## Process
1. (수동) `git status` / `git log origin/main..HEAD`로 변경·미push 확인. **push 안 하면 이전 코드로 빌드된다.** 미push 커밋이 있으면 push.
2. 메타데이터(`ios/fastlane/metadata|screenshots/**`)만 바뀌었다면 → `apply_ios_metadata`만 하고 종료(빌드 불필요).
3. `start_build` — 로컬 빌드 스크립트 + 임시 api_key 실행. `.upplz/App.ipa` 생성.
   - marketingVersion 미입력 시 MARKETING_VERSION_REQUIRED로 되물음. App Store에 이미 출시/심사된 버전(예: 1.0)이면 **더 높은 버전**(1.0.1)을 지정(train closed 예방). 빌드 번호는 자동 +1.
4. `upload_to_store` — 로컬 pilot 업로드. VERSION_ALREADY_USED면 3으로 돌아가 marketingVersion 상향.
5. `report_build_result` — 결과 보고.

## Verification
- `.upplz/App.ipa` 생성, 업로드 성공, 보고 완료.

## Next
`${CLAUDE_PLUGIN_ROOT}/references/chaining.md`에 따라 물어본다: 아이콘 갱신?(upplz-icon) / 스크린샷 갱신?(upplz-screenshot) / 메타데이터 갱신?(upplz-metadata) / 테스터 추가?(add_testers).
