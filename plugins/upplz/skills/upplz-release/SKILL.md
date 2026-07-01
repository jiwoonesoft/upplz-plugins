---
name: upplz-release
description: 빌드부터 종합 처리까지 한 번에 진행할 때 사용. 아이콘·스크린샷·메타데이터 갱신 → 빌드 → 업로드 → 테스터 → 보고를 각 단계 물어보며 순차 실행한다. "전체 배포", "한 번에 처리", "종합 배포" 요청 시 활성화.
---

## Overview
개별 스킬을 조합한 풀 체인 오케스트레이터. 최초/반복은 사전조건(금고 시크릿 유무)으로 자동 분기한다.

## When to Use
- 코드·에셋·메타를 한꺼번에 갱신하고 배포까지 끝내고 싶을 때.

## Routing (첫 단계, 필수)
`${CLAUDE_PLUGIN_ROOT}/references/routing.md` 로드. iOS ∧ 비맥이면 베타2 게이트.

## Process
각 단계를 **사용자에게 물어보며** 순차 진행(원치 않는 단계는 skip):
1. 최초 배포 여부 판단 — 셋업이 안 됐으면 `upplz-onboard` 흐름으로 위임 후 종료.
2. 아이콘 갱신? → `upplz-icon`(generate_app_icon).
3. 스크린샷 갱신? → `upplz-screenshot`.
4. 메타데이터 갱신? → `upplz-metadata`(apply).
5. 빌드→업로드 → `upplz-build`(start_build → upload_to_store). 아이콘/코드 변경이 있었다면 필수.
6. 테스터 추가? → `add_testers`.
7. `report_build_result` — 결과 보고.

## Verification
- 선택한 단계별 산출물 확인. 빌드 진행 시 `.upplz/App.ipa` + 업로드 성공 + 보고 완료.

## Next
전체 완료. 후속 개별 작업이 필요하면 해당 스킬을 안내한다.
