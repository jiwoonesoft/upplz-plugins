---
name: upplz-icon
description: 앱 아이콘을 생성하거나 교체할 때 사용. 소스 이미지로 1024 마스터 아이콘을 만들어 AppIcon.appiconset을 갱신한다(파생 사이즈는 Xcode가 빌드 시 자동 생성). "아이콘 바꿔줘", "앱 아이콘 생성", "아이콘 갱신" 요청 시 활성화.
---

## Overview
`generate_app_icon` 도구로 소스 이미지로 1024 마스터 아이콘을 생성·갱신한다(파생 사이즈는 Xcode가 빌드 시 자동 생성). 아이콘 변경은 바이너리 재빌드 후 반영된다.

소스는 두 가지 방식으로 확보할 수 있다:
- **기존 이미지 사용**: 사용자가 제공한 PNG(URL 또는 로컬 경로, 1024x1024 이상 정사각 권장).
- **새로 생성(디자인)**: 사용자가 이미지를 안 가지고 있으면, Claude가 앱 컨셉에 맞는 아이콘을 SVG로 직접 설계해 로컬 파일(`*.svg`)로 저장한 뒤 그 경로를 `iconSource`로 전달한다. 도구가 1024 PNG로 자동 래스터화한다(`rsvg-convert` 우선, 없으면 ImageMagick fallback).

## When to Use
- 앱 아이콘을 처음 넣거나 교체할 때(기존 이미지 사용 또는 새로 생성 모두 포함).

## Routing (첫 단계, 필수)
`${CLAUDE_PLUGIN_ROOT}/references/routing.md` 로드. 리사이즈는 로컬 ImageMagick(맥/윈도 무관하나, 반영을 위한 재빌드는 iOS 규칙 적용 — 비맥이면 재빌드 단계에서 베타2 게이트).

## Process
1. 소스 확보:
   - 사용자가 이미지를 제공하면 그대로 사용(URL 또는 로컬 경로, 1024x1024 이상 정사각 PNG 권장).
   - 없으면 앱에 맞는 아이콘을 SVG로 설계해 로컬에 저장하고 그 경로를 소스로 사용.
2. `generate_app_icon({ iconSource, projectType })` — 반환된 iconScript를 사용자 맥에서 실행. ImageMagick 미설치 시 `brew install imagemagick`. SVG 소스는 `rsvg-convert`(brew install librsvg) 또는 ImageMagick 중 하나가 필요.
3. appiconset 갱신 확인.

## Verification
- `AppIcon.appiconset`에 1024 마스터 아이콘 생성됨(파생 사이즈는 Xcode가 빌드 시 자동 생성).

## Next
`${CLAUDE_PLUGIN_ROOT}/references/chaining.md`: **아이콘 변경은 재빌드해야 반영된다** → upplz-build로 재빌드→업로드를 이어서 진행할지 물어본다.
