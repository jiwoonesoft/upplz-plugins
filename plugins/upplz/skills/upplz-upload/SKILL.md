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
`${CLAUDE_PLUGIN_ROOT}/references/routing.md` 로드. iOS ∧ 비맥이면 iOS 맥 전용 게이트로 중단. Android 는 맥·윈도(Git Bash + Docker Desktop) 모두 같은 Process 다.

## 최초 셋업 게이트 (두 번째 단계, 필수)
`${CLAUDE_PLUGIN_ROOT}/references/first-time-setup.md`의 "셋업 완료 판정"으로 확인한다 — **iOS·Android 모두 `get_setup_status` 로 판정**하고(iOS 는 파라미터 없이, Android 는 `platform: "android"` — 패키지명을 알면 `packageName` 도 함께 넘겨 힌트의 owner 요청문에 실제 패키지명이 찍히게 한다. 선택이며 판정은 바뀌지 않는다), `setupComplete`면 Process로, 아니면 `nextStep`으로 분기한다(두 플랫폼이 같은 모양의 루프다). 미완료면 해당 플랫폼의 최초 셋업 절차를 수행한 뒤 아래 Process로 복귀한다(사용자가 `/upplz:init`을 아직 실행하지 않았다면 그 커맨드를 권하되, 요청하면 같은 절차를 여기서 그대로 수행한다 — 둘은 같은 문서를 쓴다).

## Process
출시 목적이므로 **점검(1–3) 자체는 건너뛰지 않고 각 항목의 상태를 확인·보고**하되, 갱신 실행 여부는 사용자에게 물어본다(원치 않으면 skip). Android는 아이콘·스크린샷 자동 생성 도구가 없으므로(iOS 전용) 1–2는 상태 확인·안내만 한다. Android의 확인 대상: 아이콘은 `android/app/src/main/res/mipmap-*`, 스크린샷·스토어 이미지는 3단계 `pull_android_metadata` 결과(`android/fastlane/metadata/android/**/images`)로 확인한다.

1. **아이콘 점검** — `AppIcon.appiconset`(1024 마스터) 존재·최신 여부 확인. 갱신 필요 시 `upplz-icon`(generate_app_icon) 진행. **아이콘 변경은 바이너리 재빌드 후 반영되므로 4 이전에 수행한다.**
2. **스크린샷 점검** — `ios/fastlane/screenshots/**` 존재·현재 앱 화면과 일치 여부 확인. 갱신 필요 시 `upplz-screenshot` 진행.
3. **메타데이터 점검** — iOS는 `pull_ios_metadata`, Android는 `pull_android_metadata({ packageName })`로 현재 스토어 값을 내려받아 설명·키워드·릴리스 노트를 확인. 갱신 필요 시 `upplz-metadata`(apply) 진행. 새 버전 출시라면 릴리스 노트 갱신을 권한다. 메타데이터 도구는 활성 멤버 누구나 호출한다(협업자 포함). 자격은 스크립트가 인증서 저장소에서 풀어 쓰므로 응답 `env.MATCH_PASSWORD` 만 export 한다 — Android 의 Play Service Account 가 미등록이면 `PRECONDITION_FAILED` 이므로 owner 에게 `register_android_credentials` 를 요청한다.
4. **빌드→업로드→보고** — `upplz-upload-build-only`의 Process(iOS 1–5 / Android 1'–5')를 그대로 따른다(`repoUrl`(Android 는 `packageName` 도) 필요. 비밀값은 도구 응답 `env.MATCH_PASSWORD` 하나이며 iOS match·Android keystore·`.enc` 파일이 모두 이 값으로 열린다 — 새로 정하는 값이 아니다. 저장돼 있지 않으면 도구가 `PRECONDITION_FAILED` 로 멈추므로 owner 에게 등록을 요청한다. 역할로 갈리지 않는다). 아이콘/코드 변경이 있었다면 필수, 2–3만 갱신했고 코드·아이콘 변경이 없다면 빌드는 생략 가능(메타는 빌드와 독립).
5. (iOS 선택) 테스터 추가 → `add_testers`(선행: 4의 report_build_result로 uploaded 보고 완료. 테스터 관리는 팀 키의 ASC 역할이 App Manager 이상이어야 한다).
   - **자격 취급은 `upplz-upload-build-only` 의 스크립트 실행 규칙과 같다** — 응답 `env.MATCH_PASSWORD` 를 export 하고 스크립트를 통째로 실행하면 프롤로그가 저장소에서 `.p8` 를 풀어 쓴다. 키 파일을 직접 찾거나 `Read` 로 읽지 않는다.
6. **심사 제출 안내(수동)** — iOS: App Store Connect 웹 → 해당 버전 → Add for Review → Submit for Review. Android: Play Console 웹에서 트랙별 검토·출시. upplz 도구 범위 밖임을 안내한다.

## Verification
- 점검 항목(아이콘·스크린샷·메타데이터)별 상태가 사용자에게 보고됨. 갱신한 항목은 산출물 확인(appiconset/`ios/fastlane/screenshots/**`/apply 성공).
- 빌드 진행 시 iOS는 `.upplz/App.ipa`, Android는 `.upplz/app-release.aab` + 업로드 성공 + 보고 완료.

## Next
전체 완료. **스토어 심사 제출은 수동**임을 다시 안내한다(위 6). 이후 개별 작업이 필요하면 해당 스킬(upplz-icon/upplz-screenshot/upplz-metadata/upplz-upload-build-only)을 안내한다.
