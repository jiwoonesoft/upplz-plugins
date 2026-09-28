# upplz Claude Code 플러그인

웹 게임/앱을 iOS·Android로 빌드·배포하는 upplz 워크플로우 스킬 모음(iOS 는 맥 로컬 빌드, Android 는 맥·윈도 로컬 Docker 하네스).

## 스킬
- `upplz-upload-build-only` — 테스트용 순수 빌드→스토어 업로드(최초 배포 셋업 게이트 포함)
- `upplz-upload` — 출시용: 메타데이터·아이콘·스크린샷 점검·갱신 + 빌드→업로드(풀 체인)
- `upplz-metadata` — 스토어 메타데이터 pull/apply
- `upplz-icon` — 아이콘 생성·갱신
- `upplz-screenshot` — 스크린샷 생성·프레임

## 커맨드
- `/upplz:init` — **배포 설정 원스톱**. 서버 셋업 상태(`get_setup_status`)와 이 머신의 로컬 상태를 함께 보고 남은 설정을 끝까지 처리한다. `.p8`·Play Service Account 는 **파일 경로만** 받아(본문을 읽지 않는다) 반환된 스크립트가 인증서 저장소에 암호화해 넣는다. 협업자는 owner 전용 4단계(공유 자산 등록·초기화)만 owner 에게 요청하고 빌드·업로드·메타데이터까지 진행한다. 절차는 `references/first-time-setup.md` 하나를 upload 계열 스킬과 공유한다.

## 온보딩 순서
회원가입 → **프로젝트 생성**(대시보드) → **키 발급**(프로젝트 페이지) → 플러그인 설치·MCP 셋업 → `/upplz:init` → 배포.

## 설치

### 정식 배포 (고객)
공개 마켓플레이스 저장소를 통해 설치하면 **스킬 + 원격 MCP 연결 + API 키**가 한 번에 구성된다:

```bash
/plugin marketplace add jiwoonesoft/upplz-plugins
/plugin install upplz@upplz-tools
# 설치 중 Upplz API Key 입력(→ keychain 저장) 후:
/reload-plugins
```

퍼블리싱 담당자용 상세(전용 저장소 레이아웃·marketplace.json·버전 갱신·엔드포인트 주의)는 [DISTRIBUTION.md](./DISTRIBUTION.md) 참조.

### 개발/내부 테스트
마켓플레이스 없이 로컬 경로로 로드(`marketplace.json` 불필요, plugin.json만):

```bash
claude --plugin-dir /경로/upplz.git/plugin
/reload-plugins   # 스킬 수정 즉시 반영
```

> `--plugin-dir`는 `userConfig` 프롬프트를 안 띄우므로 번들 MCP의 API 키가 비어 인증이 안 될 수 있다 — 스킬 점검용. 전체 흐름은 `/plugin install`로 검증.

## 연결·인증
- 플러그인이 원격 upplz MCP 서버 연결을 번들한다(`plugin.json`의 `mcpServers`, Streamable HTTP).
- 인증: `Authorization: Bearer <API Key>`. API 키는 upplz 대시보드의 **프로젝트 페이지**에서 발급한다.
- **키 1개 = 프로젝트 1개.** 플러그인 설정은 설치당 키 하나만 갖는다 — 다른 프로젝트를 배포하려면 `/plugin` → upplz 설정에서 그 프로젝트의 키로 교체해야 한다(여러 프로젝트 동시 연결 불가).
- 스킬은 다른 파일을 `${CLAUDE_PLUGIN_ROOT}/references/...` 로 참조한다(플러그인은 캐시에 복사되므로 이 변수 필수).

## 전제
- upplz MCP 서버 접속(원격) — 프로젝트 식별자·비밀번호 보관과 빌드·업로드·등록 스크립트 제공(서버는 Apple·Google 을 직접 부르지 않는다).
- 인증서 저장소(owner 소유 private GitHub)의 collaborator 권한 — 인증서·프로파일·keystore·Apple 팀 키·Play Service Account 가 모두 여기 있다.
- 로컬 toolchain(**빠진 것은 에이전트가 설치한다** — 비밀번호·GUI 확인·로그인만 사용자에게 요청): iOS 는 맥의 Xcode/fastlane/openssl(ImageMagick 은 아이콘·프레임 합성 시), Android 는 Docker(`web-game-android` 이미지 — 플러그인에 든 `docker/android/Dockerfile` 로 만든다) — 윈도는 Git Bash + Docker Desktop. 절차는 `references/first-time-setup.md` 「빌드 환경 설치」.
