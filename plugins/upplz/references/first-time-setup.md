# 최초 셋업 — iOS / Android (first-time setup)

이 문서는 `/upplz:init` 커맨드와 upload 계열 스킬(`upplz-upload-build-only`·`upplz-upload`)의 최초 셋업 게이트가 **공용 참조하는 단일 절차**다(D-8). 어느 쪽으로 들어와도 같은 단계를 같은 순서로 수행한다.

셋업 미완료로 판정되면 이 문서의 해당 플랫폼 절차를 수행한 뒤 **호출한 스킬의 Process(빌드 단계)로 복귀**한다(`/upplz:init`으로 들어왔다면 커맨드의 완료 확인 단계로 복귀).

## 한 장 요약 — 무엇이 어디에 있나

- **역할은 둘이다** — `owner`(공유 자산을 만들고 바꾼다) / `collaborator`(빌드·업로드·테스터·스토어 메타데이터 전부). owner 전용 도구는 `register_ios_credentials`·`setup_match_repo`·`setup_android_keystore`·`register_android_credentials` **4종**뿐이다.
- **파일 자산은 전부 인증서 저장소(owner 소유 private GitHub 저장소, 프로젝트 시크릿 `ios/match-repo-url`) 하나에 있다** — match 인증서·프로파일, Android keystore(`android/<packageName>.keystore`), Apple API 팀 키(`upplz/ios/AuthKey.p8.enc`), Play Service Account(`upplz/android/service-account.json.enc`).
- **비밀번호는 하나다** — 프로젝트 비밀번호(`ios/match-password`). match 암호화·`.enc` 복호화·keystore 비밀번호가 **모두 이 값**이다. owner 가 최초 등록 때 `matchPassword` 로 정하고, 이후 스크립트를 돌려주는 모든 도구가 응답 `env.MATCH_PASSWORD` 로 전달한다. **저장돼 있지 않으면 도구가 `PRECONDITION_FAILED` 로 멈춘다** — 직접 입력하는 경로는 없다.
- **서버는 Apple·Google 을 부르지 않는다.** 모든 외부 호출은 사용자 머신의 스크립트가 한다. 셋업 판정은 DB 기록(시크릿 존재·audit·빌드)만 본다.
- **머신**: iOS 는 맥 전용. Android 는 맥, 그리고 윈도(Git Bash + Docker Desktop)에서 같은 스크립트를 쓴다(**윈도는 미검증** — 실측 전이다. docker 줄은 Git Bash 의 경로 자동 변환을 막도록 `MSYS_NO_PATHCONV=1`·`cygpath -m` 을 쓴다).

## 셋업 완료 판정

> **격리 단위는 프로젝트다.** Apple 식별자·인증서 저장소·프로젝트 비밀번호·Bundle ID 는 모두 **프로젝트 것 하나**이며, 같은 프로젝트의 두 역할은 **같은 판정 결과**를 본다. 역할은 판정을 바꾸지 않고 **힌트 문구와 수행 주체**만 바꾼다 — owner 전용 단계에 닿으면 협업자는 "owner 에게 요청" 안내를 본다. 셋업이 끝난 뒤는 역할로 갈리지 않는다.

- **iOS**: 추측하지 않고 **`get_setup_status`로 능동 판정**한다(read-only, 시크릿 값은 반환하지 않음).
  1. `get_setup_status` 호출(파라미터 없음 → `platform` 기본 `ios`). `bundleId` 파라미터는 **⑦ `register_bundle_id`로 이미 저장된 ID와 다른 ID를 확인할 때만** 사용한다 — 저장 전에는 넘기지 않는다(사용자가 처음 알려준 새 Bundle ID는 파라미터 없이 호출해 `nextStep: "register_bundle_id"`를 받고 ③·⑦ 절차를 따른다).
  2. `setupComplete: true` → 호출한 스킬의 Process(iOS 빌드 단계)로 바로 진행한다.
  3. **로컬 선행 단계(①–⑤)는 `nextStep`과 무관하게 첫 진입 시(§0 통과 후) 항상 먼저 수행한다** — 서버는 로컬 스캐폴드 상태를 판정하지 않으므로, `nextStep`만 보고 "해당 단계부터 수행"하면(예: 첫 사용자의 `nextStep: register_ios_credentials` → ⑥으로 바로 점프) ①–⑤(저장소 분석·빌드 환경·앱 정보·스캐폴드·push)를 건너뛰게 된다. `ios/` 디렉토리 존재 여부로 스킵 판단한다(이미 있으면 **③·④만** 생략하고 ⑤ push 여부만 확인 — ②빌드 환경 확인은 `ios/` 존재와 무관하므로 생략 대상이 아니다. **①`analyze_repository`는 read-only라 projectType 판정을 위해 항상 수행**한다 — 이미 `ios/`가 있는 native-ios/flutter 첫 빌드에서 이를 생략하면 projectType을 추측하게 된다. ③을 생략하는 경우 ⑦에 넘길 Bundle ID·앱 이름은 Xcode 프로젝트(`PRODUCT_BUNDLE_IDENTIFIER`)나 `capacitor.config`에서 읽어 사용자에게 확인만 받는다). 그 다음 `nextStep`으로 분기해 아래 "iOS 최초 셋업"의 해당 단계부터 수행하고, **`nextStepHint`를 사용자에게 그대로 안내**한다. 단계 하나를 끝낼 때마다 `get_setup_status`를 다시 호출해 다음 `nextStep`을 받는다(수동으로 순서를 추적하지 않는다).
     - `register_ios_credentials` → ⑥ / `register_bundle_id` → ③·⑦ / `setup_match_repo` → ⑧ / `start_build` → 빌드 단계(⑩).
  4. **서버가 확인하지 않는 것**: ASC 앱 레코드(⑦의 수동 단계)·프로비저닝 프로파일·인증서 저장소 안 `.enc` 파일의 실존·협업자의 저장소 접근 권한. 없으면 스크립트 실행 중에 드러난다 — 프로파일이 없으면 `start_build` 의 match 가, 앱 레코드가 없으면 업로드(pilot)가, `.enc` 가 없으면 스크립트의 `[fetch]` 가 멈추고 무엇을 해야 하는지 알려 준다.
  5. 에러 응답: `{ error: "Unauthorized" }`면 routing.md의 MCP 연결 게이트로. `SETUP_STATUS_FAILED`(서버 일시 장애)면 잠시 후 재시도.
  - 진행 중 `start_build`가 `PRECONDITION_FAILED`/`BUNDLE_ID_REQUIRED`를 반환하면 `get_setup_status`를 다시 호출해 `nextStep`으로 복귀한다.
- **Android**: 추측하지 않고 **`get_setup_status({ platform: "android" })` 로 능동 판정**한다 — iOS 와 **같은 도구·같은 모양의 응답**이고 `platform` 만 다르다. 최상위 `nextStep`·`nextStepHint`·`setupComplete` 가 Android 것으로 오므로 **iOS 와 똑같은 루프를 돈다**: `setupComplete: true` → 호출한 스킬의 Process(Android 빌드 단계)로, 아니면 `nextStep` 으로 분기하고 **`nextStepHint` 를 사용자에게 그대로 안내**한 뒤 단계를 하나 끝낼 때마다 다시 호출한다. **패키지명을 이미 알면 `packageName` 도 함께 넘긴다** — 판정은 그대로이고(선택 파라미터), 힌트의 요청문에 `setup_android_keystore({ packageName: "com.example.app" })` 처럼 **실제 패키지명이 찍혀** 협업자가 owner 에게 그 문장을 그대로 옮길 수 있다. 값은 아래 1(`analyze_repository`)·3(`setup_project` 에 넘긴 값)이나 `android/app/build.gradle` 의 `applicationId`·`capacitor.config.*` 의 `appId` 에서 온다 — **모르는 단계에서는 넘기지 않는다**(자리표시자가 그대로 나갈 뿐이다).
  1. `nextStep` 대응: `setup_android_keystore` → 아래 "Android 최초 셋업" 5 / `register_android_credentials` → 6 / `start_build` → 빌드 단계(7). `setup_android_keystore` 가 **인증서 저장소 URL 부재** 때문인지 **keystore 미준비** 때문인지는 `android.certRepoUrl` 로 구분한다(힌트가 이미 구분해서 말해 준다). **Android 전용 프로젝트의 첫 셋업**(저장소 URL·프로젝트 비밀번호가 아직 없음)은 5 의 `setup_android_keystore` 에 `matchRepoUrl`·`matchPassword` 를 함께 넘겨 셋을 한 번에 준비한다 — 힌트가 그 호출을 그대로 말하고, Play Service Account(6) 없이도 첫 AAB 를 만들 수 있다.
  2. **로컬 선행 단계(1–4)는 `nextStep`과 무관하게 첫 진입 시 항상 먼저 수행한다** — iOS ①–⑤ 와 같은 이유로, 서버는 이 머신의 스캐폴드·Docker 상태를 판정하지 않는다.
  3. **서버가 확인하지 않는 것**(판정에 넣지 않고 `nextStepHint` 로만 안내한다 — 확인할 수 없는 것을 판정에 넣으면 준비가 끝났는데도 막는 거짓 음성이 난다): ① 협업자의 **인증서 저장소 접근 권한**(owner 가 GitHub 에서 collaborator 로 추가해야 한다) ② 저장소 안 **keystore 파일의 실제 존재** — 성공한 Android 빌드가 있으면(`android.buildSucceeded: true`) 실재가 보장되지만, 없으면 `setup_android_keystore` 발급 기록(`android.keystoreScriptIssued`)으로 추정할 뿐이다. 최종 확인은 `start_build` 의 buildScript 가 하며, keystore 가 없으면 "owner 가 `setup_android_keystore` 를 실행해야 한다"며 멈춘다(그 문구를 그대로 전달한다) ③ **Play Console 앱 생성**(최초 AAB 수동 업로드 — 8). 이 셋은 `setupComplete: true` 여도 남아 있을 수 있다.
  4. 에러 응답 처리는 iOS 와 같다(`{ error: "Unauthorized" }` → routing.md MCP 연결 게이트 / `SETUP_STATUS_FAILED` → 잠시 후 재시도).

## 스크립트 실행 규칙 (모든 스크립트 도구 공통)

1. 응답의 `env.MATCH_PASSWORD`(프로젝트 비밀번호)를 사용자 셸에서 export 한다.
2. 응답의 스크립트를 **잘라내지 말고 통째로** 실행한다. 스크립트의 프롤로그가 openssl·비밀번호·저장소 접근을 확인하고(`[preflight]`), 인증서 저장소를 임시 디렉토리에 shallow clone 한 뒤(`[fetch]`), 필요한 파일만 풀어(`[decrypt]`) 쓰고, 끝나면(성공이든 실패든) `trap` 으로 임시 디렉토리를 지운다.
3. 실행 후 `unset MATCH_PASSWORD` 한다. 값을 파일·git·로그에 남기지 않는다.
4. 멈췄을 때: `[fetch] 인증서 저장소에 … 가 없습니다` → owner 가 그 메시지가 지목한 등록 도구를 실행해야 한다 / `[decrypt]` → 먼저 응답 `env.MATCH_PASSWORD` 를 그대로 export 했는지 확인하고, 맞으면 owner 가 그 파일의 등록 도구를 다시 실행해야 한다(같은 저장소를 다른 upplz 프로젝트도 쓰면 대시보드 '다른 프로젝트에서 복사' 로 비밀번호를 맞춘다 — `troubleshooting.passwordMismatch`) / `[preflight] 인증서 저장소에 접근할 수 없습니다` → 사용자 본인 git 자격 문제거나 collaborator 가 아니다(`gh auth login`, owner 에게 collaborator 추가 요청). 응답 `troubleshooting` 의 문구를 그대로 전달한다.

## iOS 최초 셋업

### §0 사전 확인 질문 (첫 진입 시 1회)

`get_setup_status`가 `setupComplete: false`이고 `apple.credentials: false`이면(= 처음 시작) 절차에 들어가기 전에 사용자에게 다음을 확인한다. **하나라도 "아니오"면 준비 방법을 안내하고 중단**한다(준비 후 다시 요청하도록).

1. **Apple Developer Program**(유료, $99/년)에 가입되어 있나요?
2. **App Store Connect API 팀 키(.p8)**를 발급받았고 **Key ID · Issuer ID · Team ID**를 알고 있나요? (ASC → Users and Access → Integrations → **Team Keys**, 역할 App Manager 이상. .p8은 발급 시 1회만 내려받을 수 있음 — 파일을 영문 경로에 두세요)
3. 인증서 보관용 **private GitHub 저장소**(인증서 저장소 = match 저장소)를 만들었고, **이 맥에서 그 저장소에 접근 가능한가요?** — `gh auth status`가 성공하거나 git HTTPS 인증(credential helper)이 되어 있어야 한다. 아니면 `gh auth login` 후 진행.

owner 가 아니면(`team.role: "collaborator"`) 1·2·3 모두 owner가 준비하므로 질문을 건너뛴다. 대신 **인증서 저장소(owner 소유 private GitHub 저장소)에 collaborator 로 추가되어 있는지**만 확인한다. 프로젝트 비밀번호는 응답 `env`로 자동 전달되므로 따로 받아 둘 필요가 없다. **App Store Connect 팀 초대는 ⑦의 앱 레코드를 협업자가 만들 때만** 필요하다(앱 생성 권한이 있는 App Manager 이상) — 업로드·테스터·메타데이터는 저장소의 팀 키로 하므로 초대가 필요 없다.

### 협업자 분기

- **owner 전용이라 멈추는 셋업 단계는 둘이다** — ⑥ `register_ios_credentials`, ⑧ `setup_match_repo`. `nextStep`이 이 중 하나를 가리키면 `nextStepHint`대로 "프로젝트 owner가 실행해야 한다"고 안내하고 owner 완료 후 재개한다.
- **나머지는 협업자가 직접 수행한다** — ①–⑤ 로컬 준비, ⑦ `register_bundle_id` + ASC 웹 앱 레코드 생성(ASC 팀 초대가 있을 때 — 없으면 owner 에게 앱 레코드만 요청), ⑨ `add_provisioning_profile`, ⑩ 빌드, 그리고 업로드·테스터·메타데이터 전부. 역할을 이유로 진행을 멈추지 말고, owner 전용 단계에 도달했을 때만 대기한다.
- owner 에게 요청할 때는 **⑥·⑧ 중 남은 것과 인증서 저장소 collaborator 추가를 한 번에 묶어** 보낸다(하나씩 보내면 왕복이 그만큼 늘어난다).
- owner 전용 도구를 호출해 `OWNER_ONLY`를 받아도 같은 안내를 한다.
- **인증서 저장소 접근 권한**: 협업자의 로컬 스크립트는 owner 소유 private GitHub 저장소를 사용자 본인 git 자격으로 clone 한다. upplz는 이 권한을 부여할 수도 확인할 수도 없다 — `nextStepHint`에 붙어 오는 안내를 **그대로 전달**한다(스크립트가 저장소 접근 오류로 멈추면 owner에게 GitHub collaborator 추가를 요청).
- owner가 upplz 도입 전이거나 upplz 밖에서 이미 match 저장소를 초기화한 경우, `nextStep: setup_match_repo`여도 무작정 대기하지 않는다. owner에게 초기화 완료 여부를 확인해 맞으면 ⑨ `add_provisioning_profile`(협업자도 호출한다)로 이 Bundle ID 의 프로파일을 받는다 — 저장소가 match 미초기화(match 브랜치 — 기본 `master` — 에 `match_version.txt` 없음)면 스크립트가 `[match]` 로 멈추므로 새 Distribution 인증서가 만들어지지 않는다(저장소에는 ⑥ 의 `.p8.enc` 가 이미 있으므로 "비어 있는가" 로는 판정하지 않는다). 확실하지 않으면 owner에게 `setup_match_repo` 재실행(멱등)을 요청해도 된다. upplz 밖에서 match 를 master 가 아닌 브랜치로 초기화했다면(Matchfile git_branch·CI 의 MATCH_GIT_BRANCH) 스크립트 실행 전에 `export MATCH_GIT_BRANCH=<그 브랜치>` 를 하세요 — 하지 않으면 add_provisioning_profile 은 [match] 로 멈추고 setup_match_repo 는 master 에 새 인증서를 발급합니다.

### 절차

1. `analyze_repository` — repoType/framework/hasCapacitor/hasMobileSupport 확인(도구가 projectType을 직접 반환하지 않는다). 이 결과와 `ios/` 디렉토리 존재 여부로 에이전트가 projectType(web-capacitor/native-ios/flutter/react-native)을 판단한다.
2. 빌드 환경 확인 — 맥에 Xcode·Command Line Tools·ruby·fastlane·openssl(맥 기본 LibreSSL 로 충분) 필요(미설치 시 설치 안내). 최종 점검은 각 스크립트의 preflight 가 수행한다.
3. **앱 정보 질문** — `get_setup_status`의 `bundleId`가 `null`이면 사용자에게 **앱 이름**과 **Bundle ID**(역도메인, 예 `com.yourname.appname`; 영숫자·하이픈·점만)를 묻는다. `bundleId`가 이미 있으면 그 값을 쓰고 묻지 않는다. 이 값이 ④·⑦·⑨·빌드에서 계속 쓰인다.
4. `setup_project` — 로컬 스캐폴드/설정(web-capacitor 전용, ③의 Bundle ID·앱 이름 사용, `iconUrl`로 초기 아이콘 지정 가능). native-ios/flutter/react-native는 건너뛴다.
5. (수동) 변경분 커밋·push — **push하지 않으면 start_build가 원격의 이전 코드로 빌드된다.**
6. **Apple 크레덴셜 등록** — `register_ios_credentials({ authKeyPath, keyId, issuerId, teamId, matchRepoUrl?, matchPassword? })` — **owner 전용**(`OWNER_ONLY`).
   - `authKeyPath` 는 `.p8` 파일의 **절대 경로**다. **파일을 `Read` 로 읽지 않는다** — 경로만 넘기면 반환되는 스크립트가 이 맥에서 파일을 확인(`openssl pkey`)하고 암호화해 인증서 저장소의 `upplz/ios/AuthKey.p8.enc` 로 push 한다. 키 본문은 대화·서버 어디에도 들어가지 않는다.
   - Key ID·Issuer ID·Team ID·저장소 URL 은 비밀이 아니므로 대화로 받는다. **`matchRepoUrl` 은 프로젝트에 아직 없을 때만(최초 등록) 넘긴다** — 키 교체·Android 를 먼저 설정한 프로젝트는 빼면 저장값이 쓰인다(넘기면 덮어쓴다). **프로젝트 비밀번호(`matchPassword`, 8자 이상)는 프로젝트에 아직 없으면 필수**다(없으면 `PRECONDITION_FAILED`). `get_setup_status` 의 `apple.matchPasswordStored: true` 면 **묻지도 넘기지도 않는다** — 저장값과 다른 값은 `PRECONDITION_FAILED` 로 거부된다(바꾸면 저장소의 모든 파일이 풀리지 않는다). 이 값이 인증서 저장소의 모든 파일을 잠그는 열쇠가 되며, 새 저장소면 지금 정한 값이 곧 match 암호화 키다(분실 시 인증서 재발급). 기존 match 저장소를 쓰면 그때 정한 값과 같아야 한다.
   - 응답의 `script` 를 "스크립트 실행 규칙" 대로 `env.MATCH_PASSWORD` 와 함께 실행한다. 같은 파일로 다시 실행하면 `[done] 변경 없음` 으로 끝난다.
   - **키 교체 = 같은 도구 재실행.** 응답 `replaced` 는 **저장돼 있던 Key ID 가 이번 값과 다를 때만 true** 다(같은 키 재등록·기존 프로젝트 이관은 false). true 면 응답 `previousKeyId` 를 App Store Connect ▸ Team Keys 에서 Revoke 하라고 안내한다 — 스크립트가 저장소 파일을 새 키로 바꾼 **뒤에** 한다.
   - 형식 오류는 `INVALID_INPUT`의 `errors`(필드별 안내)로 오므로 해당 값만 다시 받아 재호출한다. 입력값은 응답·로그 어디에도 되돌아오지 않는다.
   - owner 가 아니면 `OWNER_ONLY`로 막힌다 — "협업자 분기" 안내.
7. `register_bundle_id({ bundleId, appName })` — 활성 멤버 누구나. 반환된 로컬 `fastlane produce` 스크립트를 실행하면(스크립트가 저장소에서 `.p8` 를 풀어 쓴다) Developer Portal 에 Bundle ID 가 등록된다.
   - **그 다음 ASC 웹에서 앱 레코드를 직접 만든다(수동)** — App Store Connect ▸ My Apps ▸ + ▸ New App ▸ Bundle ID 드롭다운에서 ⑦의 ID 선택. App Store Connect API 는 앱 생성을 지원하지 않아 자동화할 수 없고, 서버도 확인하지 않는다(없으면 업로드 때 pilot 이 알려 준다). 만드는 사람은 ASC 팀에 **App Manager 이상**이어야 한다. 앱 레코드는 업로드 전에만 있으면 되므로, 기다리는 동안 ⑧로 가도 된다.
   - **멱등 스킵**: produce 스크립트가 `already exists, nothing to do` 류 메시지를 출력하며 정상 종료(exit 0)하면 이미 등록된 것으로 간주한다.
   - **Bundle ID 충돌**: 스크립트가 `not available`/`cannot be registered to your development team`으로 실패하면 다른 Apple 팀이 점유 중이다(응답 `troubleshooting.takenByOtherTeam`). **③ 앱 정보 질문으로 복귀**해 다른 Bundle ID를 받고 ④(스캐폴드의 Bundle ID 교체)부터 다시 진행한다.
   > (선택) 초기 메타데이터·스크린샷이 필요하면 앱 레코드 생성 후부터 가능하다 — upplz-upload의 점검 단계 또는 upplz-metadata/upplz-screenshot 스킬 참조.
8. `setup_match_repo({ matchRepoUrl?, forceNuke? })` — **owner 전용**(`OWNER_ONLY`). match 저장소 초기화 + 공용 Distribution 인증서(최초 1회만 발급, 이후 재사용) + 이 Bundle ID 의 첫 프로비저닝 프로파일을 준비하는 로컬 스크립트. **비밀번호는 ⑥에서 저장됐다** — 이 도구는 비밀번호를 받지 않고 저장값을 `env.MATCH_PASSWORD` 로 싣는다. 스크립트가 실행 전 **fastlane 설치·저장소 접근**을 점검(preflight)하고, clone 뒤 **match 초기화 여부**(match 가 쓰는 브랜치 — `MATCH_GIT_BRANCH`, 기본 `master` — 의 `match_version.txt`)로 새 저장소인지 기존 저장소인지 알려 주며, 실패 원인은 응답 `troubleshooting`(`fastlaneMissing`/`repoAccessDenied`/`passwordMismatch`/`certLimit`)으로 안내한다. 응답의 `matchPassword`는 정책 안내 객체(`policy`/`firstTime`/`existing`)이며 **값은 포함되지 않는다** — 값은 `env.MATCH_PASSWORD` 로만 온다.
   - **멱등 스킵**: `get_setup_status`의 `apple.matchInitialized: true`면 스크립트가 발급된 기록이 있다는 뜻이다(실행 성공을 보증하지 않는 약한 근거). 스크립트를 실제로 실행해 성공했는지 사용자에게 확인하고, 안 했으면 이 단계를 다시 수행한다(멱등).
9. (필요할 때만) `add_provisioning_profile({ bundleId? })` — 활성 멤버 누구나. **다른 Bundle ID**(익스텐션·두 번째 앱)의 프로파일을 저장소에 추가한다(인증서는 ⑧ 결과를 재사용). 저장소가 **match 미초기화(match 브랜치에 `match_version.txt` 없음)면 스크립트가 `[match]` 오류로 멈춘다**(owner 가 ⑧을 아직 안 했다는 뜻 — 협업자가 read-write match 로 새 Distribution 인증서를 만들어 버리는 것을 막는다). ASC 앱 레코드는 필요 없다(프로파일은 Developer Portal 의 Bundle ID 로 만든다). match 는 멱등이라 같은 ID 로 다시 실행해도 안전하다. 접근 실패·비밀번호 불일치는 응답 `troubleshooting`(`repoAccessDenied`/`passwordMismatch`)을 참조한다.
10. `get_setup_status`로 `setupComplete: true` 확인 → 호출한 스킬의 Process(iOS 빌드 단계)로 복귀(`/upplz:init`으로 들어왔다면 커맨드의 완료 확인 단계로). **`start_build` 응답에는 Apple API 키(.p8)가 들어 있지 않다** — buildScript 의 match 가 `--readonly` 라 저장소의 인증서·프로파일만 받고 Apple 을 치지 않는다. 응답 `env` 는 `MATCH_PASSWORD` 하나다. 첫 빌드 시 native-ios/flutter/react-native는 `projectType`을 반드시 명시한다(미지정 시 web-capacitor로 잘못 빌드됨. 빌드 대상이 모호하면 `scheme`/`xcodeProjectPath` 지정). 프로파일이 없어 match 가 실패하면 ⑨로 그 Bundle ID 의 프로파일을 추가한 뒤 다시 빌드한다.

## Android 최초 셋업 (맥 · 윈도)

Android는 Apple 크레덴셜과 분리된 경로다. 빌드·업로드는 사용자 머신의 Docker(`web-game-android` 이미지) 안에서만 돈다. **윈도는 Git Bash(Git for Windows 의 bash·openssl) + Docker Desktop** 에서 맥과 같은 스크립트를 실행한다(**미검증** — 윈도 실측 전이다. docker 줄은 Git Bash 의 경로 자동 변환을 막도록 `MSYS_NO_PATHCONV=1` 을 붙이고 호스트 경로를 `cygpath -m` 으로 바꾼다).

**Android 자격은 전부 인증서 저장소에 있다** — keystore **파일**은 `android/<packageName>.keystore`, **Play Service Account JSON** 은 `upplz/android/service-account.json.enc`(프로젝트 비밀번호로 암호화)다. keystore **비밀번호는 프로젝트 비밀번호와 같은 값**이라 따로 정하거나 저장하지 않는다. 프로젝트 DB 에는 SA 의 `client_email`(공개 식별자)만 남는다 — 이 값의 존재가 "SA 등록됨" 이다.

> **Android 도 역할 선이 iOS 와 같다.** 공통 조건은 두 가지 — ① 인증서 저장소(owner 소유 private GitHub)의 **collaborator** 여야 하고(owner 가 GitHub 에서 직접 추가한다 — upplz 가 부여할 수 없다) ② 활성 멤버십이어야 한다. 그 위에서 **협업자도 빌드(7)·업로드(9)·스토어 메타데이터까지** 간다. 셋업의 owner 전용은 아래 **5(`setup_android_keystore`)와 6(`register_android_credentials`)** 뿐이며, 그 둘만 owner 에게 요청하고 대기한다.
>
> **인증서 저장소는 플랫폼 공통이다.** 시크릿 키 이름이 `ios/match-repo-url` 이지만 Android 도 같은 값을 읽는다 — iOS 를 이미 설정했다면 같은 저장소가 자동으로 쓰이고, Android 전용 프로젝트만 `matchRepoUrl` 파라미터로 한 번 등록하면 된다(`ios/` 접두사는 역사적 흔적이다).

1. `analyze_repository` — repoType/framework/hasCapacitor/hasMobileSupport 확인(도구가 projectType을 직접 반환하지 않는다). 이 결과와 `android/` 디렉토리 존재 여부로 에이전트가 projectType(web-capacitor/native-android)을 판단한다.
2. Docker 이미지 준비 확인 — `web-game-android` 이미지가 없으면 **`docker build --platform linux/amd64 -t web-game-android docker/android/`** 로 빌드하도록 안내한다. Apple Silicon 맥에서도 반드시 `--platform linux/amd64` 로 만든다 — 스크립트가 `--platform linux/amd64` 로 실행하므로, arm64 로 빌드된 옛 이미지는 맞는 이미지를 찾지 못해 pull 을 시도하다 실패한다. 최종 점검은 `start_build` 응답의 preflightScript 가 수행한다.
3. `setup_project({ platform: "android", bundleId: packageName, packageName, appName })` — `bundleId`는 zod 스키마상 항상 필수(빠뜨리면 도구 호출이 거부됨)이므로 Android에서도 반드시 전달해야 하며, 보통 `packageName`과 동일한 값을 넣는다. Capacitor Android 플랫폼 스캐폴드(web-capacitor 전용, `packageName` 미지정 시 `bundleId` 값을 패키지명으로 대체 — `packageName` 명시를 권장). native-android는 이미 `android/`가 있으므로 건너뛴다. **packageName에 언더스코어(`_`)가 포함되면 `bundleId` 필드 형식(하이픈만 허용)과 달라 그대로 넣을 수 없음 — 이 경우 `bundleId`에는 언더스코어를 뺀 유효 값을 넣고 `packageName`을 별도 지정한다.**
4. (수동) 변경분 커밋·push — **push하지 않으면 start_build가 원격의 이전 코드로 빌드된다.**
5. `setup_android_keystore({ packageName, matchRepoUrl?, matchPassword? })` (**owner 전용**) — keystore 를 **인증서 저장소에 준비**하는 로컬 스크립트. 스크립트는 사용자 본인 git 자격으로 저장소를 받아 **3분기**로 동작한다: ① 저장소에 `android/<packageName>.keystore` 가 있으면 **프로젝트 비밀번호로 열어 보고 재사용** ② 이 머신에 옛 로컬 keystore 가 있으면 내용 그대로 저장소로 **이관**해 push ③ 둘 다 없으면 Docker 의 keytool 로 프로젝트 비밀번호를 써서 **생성**해 push. **기존 파일은 어느 경우에도 덮어쓰지 않는다** — Play 앱 서명(Play App Signing)을 쓰는 앱에서 이 keystore 는 **업로드 키**라 분실·유출 시 Play Console 의 업로드 키 재설정으로 **교체할 수 있지만**(앱 서명 키는 Google 이 보관), 재설정에는 Google 지원 절차와 시간이 들고 그동안 업로드가 막히며, **Play 앱 서명을 쓰지 않는 옛 앱**이라면 이 keystore 가 곧 앱 서명 키라 **교체 불가**다.
   - keystore 비밀번호 = 프로젝트 비밀번호다(따로 정하지 않는다). **프로젝트 비밀번호가 아직 없으면(Android 전용 첫 셋업) 이 도구의 `matchPassword`(8자 이상)로 정한다** — 없이 부르면 `PRECONDITION_FAILED`. 이미 저장돼 있으면(`apple.matchPasswordStored: true`) 넘기지 않는다(저장값과 다른 값은 거부된다).
   - `matchRepoUrl` 은 미입력 시 프로젝트의 `ios/match-repo-url`(iOS 와 공용)을 쓰고, 그것도 없으면 `PRECONDITION_FAILED`. URL·비밀번호는 **모든 검증을 통과한 뒤에만** 저장된다(실패한 호출이 URL 만 남기지 않는다).
   - **옛 keystore 가 다른 비밀번호로 만들어졌다면** 스크립트가 `[error] … 열리지 않습니다` 로 멈추고 push 하지 않는다. owner 가 스크립트가 출력한 명령으로 **한 번만** 비밀번호를 프로젝트 비밀번호로 바꾼 뒤 이 도구를 다시 실행한다. 저장소 쪽 keystore 라면(저장소를 clone 한 디렉토리에서 `OLD=<옛 비밀번호>` 와 `MATCH_PASSWORD` 를 export 한 뒤) 출력되는 명령은 이 한 줄이다:
     `MSYS_NO_PATHCONV=1 docker run --rm --platform linux/amd64 -e OLD -e MATCH_PASSWORD -v "$(cygpath -m "$PWD/android" 2>/dev/null || echo "$PWD/android")":/keystores web-game-android keytool -storepasswd -keystore /keystores/<packageName>.keystore -storepass:env OLD -new:env MATCH_PASSWORD`
     (`MSYS_NO_PATHCONV=1`·`cygpath` 는 윈도 Git Bash 에서 경로 변환을 막는 장치이고 맥에서는 아무 영향이 없다.) 그 뒤 `android/<packageName>.keystore` 를 commit·push 한다. PKCS12 는 이 한 명령으로 키 엔트리까지 바뀌며 `-keypasswd` 는 거부된다(키 자체는 그대로라 Play 업로드 키가 바뀌지 않는다). 프로젝트 비밀번호 쪽을 바꾸면 저장소의 다른 파일이 전부 안 풀리므로 **keystore 쪽을 맞춘다**.
   - **owner 가 아니면 이 단계를 수행하지 않는다**(`OWNER_ONLY`) — owner 에게 요청하고 **이 단계만 대기**한다.
     - **요청은 5·6을 한 번에 묶어서 한다.** 6(`register_android_credentials`)도 owner 전용이라, 5만 요청하면 빌드까지 간 뒤 업로드에서 `PRECONDITION_FAILED` 로 또 막혀 왕복이 두 번 된다. owner 에게 보낼 요청에 **① `setup_android_keystore({ packageName })` ② `register_android_credentials({ serviceAccountPath, clientEmail })` ③ 인증서 저장소 collaborator 추가**를 함께 적는다. 이 요청문은 `nextStepHint` 가 이미 만들어 준다 — `get_setup_status` 에 `packageName` 을 넘겼다면 **실제 패키지명까지 찍혀** 있으므로 그 문장을 **그대로 옮기면 된다**(문구를 다시 쓰지 말 것). 넘기지 않았다면 owner 가 되묻지 않도록 패키지명을 직접 채워 보낸다.
     - owner 가 끝내면 **7(첫 빌드)부터 재개**한다. 5·6 중 무엇이 남았는지는 `get_setup_status({ platform: "android", packageName })` 의 `nextStep` 이 알려 준다 — `setup_android_keystore` 면 5, `register_android_credentials` 면 6, `start_build` 면 둘 다 끝났다는 뜻이다(협업자도 owner 와 **같은 판정 결과**를 본다). 그 뒤에도 keystore 파일 실존은 `start_build` 의 buildScript 가, SA 파일 실존은 업로드·메타데이터 스크립트의 `[fetch]` 가 최종 확인한다.
6. (수동 발급 → 등록) Google Play Developer 계정($25 일회성, 승인 최대 48시간) + Google Cloud **Service Account 키(JSON)** 발급 + Play Console에서 해당 SA에 API 권한 부여. 발급이 끝나면 **`register_android_credentials({ serviceAccountPath, clientEmail, matchRepoUrl?, matchPassword? })`(owner 전용)** 로 등록한다.
   - `serviceAccountPath` 는 키 파일의 **절대 경로**다. **파일을 `Read` 로 읽지 않는다.** `clientEmail` 은 `jq -r .client_email <파일>`(없으면 `node -e 'console.log(require(process.argv[1]).client_email)' <파일>`)로 뽑는다 — 이메일 한 줄만 출력되므로 키 본문이 대화에 올라오지 않는다.
   - 응답의 `script` 를 "스크립트 실행 규칙" 대로 실행하면 스크립트가 파일이 Service Account 키이고 `client_email` 이 일치하는지 확인한 뒤 암호화해 `upplz/android/service-account.json.enc` 로 push 한다. 교체라면 옛 `private_key_id` 를 출력하므로 Google Cloud 에서 그 키를 삭제하라고 안내한다.
   - 5 에서 저장소 URL·프로젝트 비밀번호를 이미 정했다면 **둘 다 빼고** 부른다(비밀번호는 저장값과 다르면 거부된다). SA 를 keystore 보다 먼저 등록하는 경우에만 `matchRepoUrl`·`matchPassword` 를 함께 넘긴다.
   - 등록하면 `upload_to_store(platform=android)`·`apply_android_metadata`·`pull_android_metadata` 의 스크립트가 저장소에서 이 파일을 풀어 쓴다. 미등록 상태로 그 셋을 호출하면 `PRECONDITION_FAILED`("owner가 `register_android_credentials`로 등록해야 합니다")가 온다.
   - 원본 키 파일은 등록 후 이 머신에 둘 필요가 없다(저장소에 암호화돼 있다).
   - 또한 Play 정책상 **API 업로드는 최초 1회 Play Console 웹에서 AAB를 수동 업로드해 앱을 생성한 뒤부터만 가능**하다는 점을 미리 안내한다(실제 수행은 8).
7. → 호출한 스킬의 Process(Android 빌드 단계)로 복귀해 첫 빌드로 `.upplz/app-release.aab`를 생성한다. `start_build({ platform: "android" })` 응답의 `env.MATCH_PASSWORD` 가 keystore 비밀번호로도 쓰인다.
8. (수동, 최초 1회만) **Play Console 웹 → 앱 만들기 → 생성된 `.upplz/app-release.aab`를 내부 테스트 트랙에 직접 업로드해 앱을 완성한다.** fastlane supply(API 업로드)는 앱이 이미 존재해야 동작하므로 자동화할 수 없다 — 최초 배포에서는 이 수동 업로드가 `upload_to_store`를 대체하며, `upload_to_store`는 반복 빌드부터 사용한다.
9. **반복 빌드부터** `upload_to_store({ platform: "android", ... })` 를 쓴다 — 활성 멤버 누구나. 응답의 `env.MATCH_PASSWORD` 를 export 한 뒤 스크립트를 **통째로** 실행하면, 프롤로그가 저장소에서 SA 를 임시 파일(0600)로 풀고 컨테이너에는 **값이 아니라 마운트된 경로**(`SUPPLY_JSON_KEY`)만 넘긴 뒤 종료 시 지운다. `apply_android_metadata`·`pull_android_metadata` 도 같은 방식이다.
