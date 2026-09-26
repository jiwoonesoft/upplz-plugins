# 라우팅 — 머신(맥/윈도) × 대상(iOS/Android)

모든 upplz 스킬은 첫 단계에서 이 문서를 로드해 **머신**과 **대상**을 확정한 뒤 진행한다.

## 머신 감지

빌드가 실행되는 곳은 **사용자 로컬 머신**(Claude Code가 도는 환경)이다. 원격 MCP 서버 OS가 아니다.

- `uname -s` → `Darwin`이면 맥, `MINGW*`/`MSYS*`면 윈도(Git Bash), 그 외는 지원하지 않는다.
- 불확실하면 사용자에게 "지금 빌드를 돌릴 이 컴퓨터가 맥인가요, 윈도인가요?"로 확인한다.

## 대상 감지

저장소 구조로 판정(또는 사용자 확인):
- `ios/` 존재 + 웹/Capacitor/Flutter/RN → **iOS** 대상 가능.
- `android/` 또는 안드로이드 빌드 요청 → **Android** 대상.

## 2×2 분기

| 대상 \ 머신 | 맥 (로컬) | 윈도 (로컬) |
|---|---|---|
| **iOS** | ✅ 로컬 빌드 | 🚧 미지원(후속 — 맥 전용) |
| **Android** | ✅ 로컬 Docker 하네스 | ⚠️ 로컬 Docker 하네스(Git Bash + Docker Desktop, 맥과 같은 스크립트 — **미검증**: 윈도 실측 전) |

## 게이트 규칙

- **MCP 연결 게이트(최우선)**: 첫 upplz MCP 도구 호출이 연결 실패/인증 실패하면, 플러그인 설정의 `Upplz MCP Endpoint`가 기본 플레이스홀더(`https://YOUR-UPPLZ-ENDPOINT`)로 남아있지 않은지, API 키가 입력됐는지 확인하도록 안내하고 **중단**한다(재설정: `/plugin` → upplz 설정, 값은 upplz 대시보드/온보딩 안내 참조).
  - 401 응답에 `reason`/`hint`가 있으면 그대로 안내한다: `missing_header`(키 미입력) / `malformed_header`(Bearer 형식 아님·공백 등) / `invalid_or_inactive`(키가 틀렸거나 **재발급·프로젝트 전환·멤버십 회수로 폐기됨** — 대시보드 **프로젝트 페이지**에서 키를 재발급했다면 이전 키는 즉시 비활성이므로 `/plugin` → upplz 구성에서 새 키로 갱신. 키 1개는 프로젝트 1개에 묶이므로 다른 프로젝트를 배포하려면 그 프로젝트의 키로 교체해야 한다). `reason`이 없는 구형 응답은 위 세 가지를 모두 점검하도록 안내한다.
  - 500 응답이면 인증 처리 중 서버 내부 오류(예: DB 장애)다 — 사용자 설정 문제가 아니므로 잠시 후 재시도하도록 안내한다.
- **셋업 게이트(iOS·Android)**: MCP 연결이 확인되면 **첫 MCP 도구 호출**은 `get_setup_status({ platform })` 으로 한다(iOS 는 파라미터 없이 호출해도 같다 — 미지정 기본이 `ios`. Android 는 패키지명을 이미 알면 `packageName` 도 함께 넘겨 힌트의 요청문에 찍히게 한다 — 선택이며 판정은 바뀌지 않는다). `setupComplete`면 빌드 단계, 아니면 `nextStep`으로 분기(`first-time-setup.md` "셋업 완료 판정"). 로컬 선행 단계는 이 판정 직후 먼저 수행한다 — iOS 는 ①`analyze_repository`~⑤ push(`analyze_repository` 자체는 MCP 도구이지만 `get_setup_status` 판정과는 별개로 그 다음에 순서대로 진행하는 로컬 체크리스트다), Android 는 `first-time-setup.md` "Android 최초 셋업" 1–4.
  - **최초 셋업의 명시적 진입점은 `/upplz:init` 커맨드**이며, 동일 절차(`first-time-setup.md`)를 수행한다. 사용자가 아직 실행하지 않았다면 그 커맨드를 권하되, 스킬 안에서 이어서 진행해 달라고 하면 같은 절차를 여기서 그대로 수행한다.
- **대상=iOS ∧ 머신≠맥** → 다음을 안내하고 **중단**한다:
  > "현재 upplz는 iOS를 **맥 로컬 빌드**로만 지원합니다. 윈도에서의 iOS 빌드는 후속으로 제공될 예정입니다. 맥에서 다시 시도해 주세요(Android 는 이 윈도 머신에서 진행할 수 있습니다 — 윈도 Android 는 아직 실측 전입니다)."
  <!-- 후속 삽입 지점: 윈도 iOS(CI 러너) 경로를 여기에 추가한다. -->
- **대상=iOS ∧ 머신=맥** → 정상 경로. 각 스킬의 Process 진행.
- **대상=Android ∧ 머신=맥** → 정상 경로. `setup_android_keystore`(owner 전용) → `register_android_credentials`(owner 전용) → `start_build({ platform: "android" })` → `upload_to_store({ platform: "android" })` 체인을 사용자 맥 로컬 Docker(`web-game-android` 이미지)에서 실행한다. 각 스킬의 Process(Android) 진행. **협업자도 빌드·업로드·스토어 메타데이터까지** 직접 수행한다 — 조건은 인증서 저장소(keystore·Play Service Account 가 사는 private GitHub 저장소)의 collaborator + 활성 멤버십이다. Apple Silicon 맥에서는 이미지를 반드시 `docker build --platform linux/amd64 -t web-game-android docker/android/` 로 만든다(스크립트가 `--platform linux/amd64` 로 실행한다).
- **대상=Android ∧ 머신=윈도** → 맥과 **같은 경로·같은 스크립트**다(**미검증** — 윈도 실측 전이다. docker 줄은 Git Bash 의 경로 자동 변환을 막도록 `MSYS_NO_PATHCONV=1` 을 붙이고 호스트 경로를 `cygpath -m` 으로 바꾼다. 스크립트가 경로 오류로 멈추면 출력을 그대로 사용자에게 보여 주고 upplz 에 알려 달라고 안내한다). 전제는 셋:
  1. **Git Bash**(Git for Windows) — 스크립트는 bash 이고, 인증서 저장소의 `.enc` 를 푸는 `openssl` 도 Git for Windows 에 들어 있다. 스크립트는 반드시 Git Bash 에서 실행한다(PowerShell·cmd 아님).
  2. **Docker Desktop**(Linux 컨테이너 모드) — 빌드·업로드·메타데이터가 전부 `web-game-android` 컨테이너에서 돈다. 이미지는 `docker build --platform linux/amd64 -t web-game-android docker/android/` 로 만든다.
  3. **Node.js** — owner 가 `register_android_credentials` 를 실행할 때 스크립트가 `node` 로 키 파일을 확인한다(`jq` 는 필요 없다).
  - 파일 경로 파라미터(`serviceAccountPath`)는 Git Bash 형식의 절대 경로로 넘긴다(예: `/c/Users/me/Downloads/my-project-1a2b3c.json`). 사용자 이름이 한글이면(`/c/Users/홍길동/…`) 도구가 거부하므로 `/c/upplz/` 같은 사용자 폴더 밖 영문 폴더로 옮긴다.
