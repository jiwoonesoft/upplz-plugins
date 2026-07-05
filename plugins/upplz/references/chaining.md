# 스킬 연속(체이닝) 규칙

각 스킬은 작업 완료 후 `## Next` 섹션에서 후속 스킬을 **사용자에게 물어보고** 연결한다(자동 강제 아님, opt-in).

## 연속 그래프

- **upplz-upload-build-only** 완료 → 제안: 출시까지 진행?(upplz-upload — 메타·아이콘·스샷 점검 포함) / 테스터 추가?(add_testers — iOS 전용, report_build_result로 uploaded 보고가 끝난 빌드만) / 아이콘·스크린샷·메타데이터 개별 갱신?(upplz-icon·upplz-screenshot — iOS 전용 / upplz-metadata)
- **upplz-upload** 완료 → 심사 제출 수동 안내 후 종료(풀 체인이므로 추가 제안 없음).
- **upplz-icon** 완료 → 제안: **재빌드 필요** → upplz-upload-build-only(아이콘 변경은 바이너리 재빌드 후 반영, iOS 전용). 출시 흐름(upplz-upload) 중이었다면 그 흐름으로 복귀해 나머지 점검(2–3)을 마저 진행하고 재빌드는 4단계에서 수행.
- **upplz-screenshot** 완료 → 제안: 메타데이터 반영?(apply, upplz-metadata) / 재빌드?(upplz-upload-build-only) — iOS 전용
- **upplz-metadata** 완료 → 독립 종료(빌드와 무관). 코드 변경이 있으면 upplz-upload-build-only 제안. Android는 apply_android_metadata만 해당.

## Android 체인 (베타1: 맥 전용)

Android는 `generate_app_icon`/`generate_ios_screenshots`/`generate_native_screenshots`/`add_testers` 대상이 아니다(모두 iOS 전용 도구). Android 체인은 다음으로 좁혀진다:

- **upplz-upload-build-only(Android)** 완료(최초 배포면 first-time-setup.md의 Android 절차: `setup_project(android)` → `setup_android_keystore` → (사전 안내: Play 개발자 계정/Service Account) → 첫 빌드로 AAB 생성 → Play Console 앱 생성+최초 AAB 수동 업로드(최초 1회만) 후 반복 빌드부터 `upload_to_store(android)`) → 제안: 출시까지 진행?(upplz-upload) / 메타데이터 정리?(upplz-metadata → `apply_android_metadata`, 선행: `pull_android_metadata`로 기존 값 확인 가능)
- **upplz-upload(Android)** → 아이콘/스크린샷은 상태 확인·안내만, 메타데이터 점검 → 빌드 → 업로드 → 보고 → 심사 제출 수동 안내.
- **upplz-metadata(Android)** 완료 → 독립 종료(빌드와 무관). 코드 변경이 있으면 upplz-upload-build-only(Android) 제안.

## 제안 문구 규칙

- 항상 "지금 이어서 ~를 진행할까요?" 형태로 **명시적으로 질문**하고 사용자 응답을 기다린다.
- 사용자가 원치 않으면 종료. 강제로 다음 스킬을 실행하지 않는다.
- 재빌드가 반드시 필요한 경우(아이콘 변경)는 "변경 반영에는 재빌드가 필요합니다"를 명확히 알린다.

## 마무리 안내 (공통)

- 업로드 성공 후 Apple 처리(5–30분)가 끝나야 TestFlight에서 빌드가 보인다 — 대기 안내.
- **스토어 심사 제출은 항상 수동**: App Store Connect 웹 → 해당 버전 → Add for Review → Submit for Review. 어느 스킬도 자동 제출하지 않는다.
