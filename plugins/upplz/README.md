# upplz Claude Code 플러그인

웹 게임/앱을 iOS로 빌드·배포하는 upplz 워크플로우 스킬 모음(베타1: 맥 로컬 빌드).

## 스킬
- `upplz-onboard` — 최초 배포(Apple 셋업 → 첫 빌드/업로드)
- `upplz-build` — 반복 빌드→업로드→보고
- `upplz-metadata` — 스토어 메타데이터 pull/apply
- `upplz-icon` — 아이콘 생성·갱신
- `upplz-screenshot` — 스크린샷 생성·프레임
- `upplz-release` — 빌드부터 종합처리(풀 체인)

## 설치

### 정식 배포 (고객)
공개 마켓플레이스 저장소를 통해 설치하면 **스킬 + 원격 MCP 연결 + API 키**가 한 번에 구성된다:

```bash
/plugin marketplace add jiwoonesoft/upplz-plugins
/plugin install upplz@upplz-tools
# 설치 중 Upplz API Key 입력(→ keychain 저장) 후:
/reload-plugins
```

배포자용 상세(전용 저장소 레이아웃·marketplace.json·버전 갱신·엔드포인트 주의)는 [DISTRIBUTION.md](./DISTRIBUTION.md) 참조.

### 개발/내부 테스트
마켓플레이스 없이 로컬 경로로 로드(`marketplace.json` 불필요, plugin.json만):

```bash
claude --plugin-dir /경로/upplz.git/plugin
/reload-plugins   # 스킬 수정 즉시 반영
```

> `--plugin-dir`는 `userConfig` 프롬프트를 안 띄우므로 번들 MCP의 API 키가 비어 인증이 안 될 수 있다 — 스킬 점검용. 전체 흐름은 `/plugin install`로 검증.

## 연결·인증
- 플러그인이 원격 upplz MCP 서버 연결을 번들한다(`plugin.json`의 `mcpServers`, Streamable HTTP).
- 인증: `Authorization: Bearer <API Key>`. API 키는 upplz 대시보드 설정 페이지에서 발급.
- 스킬은 다른 파일을 `${CLAUDE_PLUGIN_ROOT}/references/...` 로 참조한다(플러그인은 캐시에 복사되므로 이 변수 필수).

## 전제
- upplz MCP 서버 접속(원격) — 금고·Apple 오케스트레이션·빌드 스크립트 제공.
- 맥 로컬: Xcode/fastlane/ImageMagick 등 빌드 toolchain(스킬이 설치 안내).
