# 스킬 연속(체이닝) 규칙

각 스킬은 작업 완료 후 `## Next` 섹션에서 후속 스킬을 **사용자에게 물어보고** 연결한다(자동 강제 아님, opt-in).

## 연속 그래프

- **upplz-onboard** 완료 → 제안: 스크린샷 생성?(upplz-screenshot) / 메타데이터 정리?(upplz-metadata) / 테스터 추가?(add_testers — report_build_result로 uploaded 보고가 끝난 빌드만)
- **upplz-build** 완료 → 제안: 아이콘 갱신?(upplz-icon) / 스크린샷 갱신?(upplz-screenshot) / 메타데이터 갱신?(upplz-metadata) / 테스터 추가?(add_testers — report_build_result로 uploaded 보고가 끝난 빌드만)
- **upplz-icon** 완료 → 제안: **재빌드 필요** → upplz-build (아이콘 변경은 바이너리 재빌드 후 반영)
- **upplz-screenshot** 완료 → 제안: 메타데이터 반영?(apply, upplz-metadata) / 재빌드?(upplz-build)
- **upplz-metadata** 완료 → 독립 종료(빌드와 무관). 코드 변경이 있으면 upplz-build 제안.
- **upplz-release** → 위 전부를 각 단계 물어보며 순차 진행하는 풀 체인.

## 제안 문구 규칙

- 항상 "지금 이어서 ~를 진행할까요?" 형태로 **명시적으로 질문**하고 사용자 응답을 기다린다.
- 사용자가 원치 않으면 종료. 강제로 다음 스킬을 실행하지 않는다.
- 재빌드가 반드시 필요한 경우(아이콘 변경)는 "변경 반영에는 재빌드가 필요합니다"를 명확히 알린다.

## 마무리 안내 (공통)

- 업로드 성공 후 Apple 처리(5–30분)가 끝나야 TestFlight에서 빌드가 보인다 — 대기 안내.
- **스토어 심사 제출은 항상 수동**: App Store Connect 웹 → 해당 버전 → Add for Review → Submit for Review. 어느 스킬도 자동 제출하지 않는다.
