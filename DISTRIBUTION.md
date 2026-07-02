# upplz 플러그인 정식 배포 가이드

고객에게 `upplz` 플러그인을 배포하는 방법. 설치 한 번으로 **스킬 6개 + 원격 MCP 서버 연결 + API 키 구성**이 함께 완료된다.

## 배포 채널: 공개 플러그인 전용 저장소

서버 코드와 분리된 **별도 공개 git 저장소** `upplz-plugins`로 배포한다(플러그인만 담기므로 공개해도 서버 노출 없음).

### 전용 저장소 레이아웃

```
upplz-plugins/                       # 공개 git 저장소 (마켓플레이스)
├── .claude-plugin/
│   └── marketplace.json             # 마켓플레이스 카탈로그
└── plugins/
    └── upplz/                       # 이 저장소의 plugin/ 내용을 그대로 복사
        ├── .claude-plugin/plugin.json
        ├── skills/
        ├── references/
        ├── README.md
        └── DISTRIBUTION.md
```

`plugins/upplz/`는 본 저장소(upplz.git)의 `plugin/` 디렉토리 내용을 그대로 옮긴 것이다. 릴리스마다 이 디렉토리를 동기화하고 `plugin.json`의 `version`을 올린다.

### marketplace.json

```json
{
  "name": "upplz-tools",
  "owner": { "name": "Upplz" },
  "description": "웹 게임/앱을 iOS로 빌드·배포하는 upplz 플러그인",
  "plugins": [
    {
      "name": "upplz",
      "source": "./plugins/upplz",
      "description": "온보딩·빌드·메타데이터·아이콘·스크린샷·종합처리 + 원격 MCP 연결 번들",
      "version": "0.1.3"
    }
  ]
}
```

> `marketplace.json` 스키마(특히 `source` 표기)는 배포 직전 [공식 문서](https://code.claude.com/docs/en/plugin-marketplaces.md)로 최종 확인할 것.

## 고객 설치 절차

```bash
# 1. 마켓플레이스 추가
/plugin marketplace add jiwoonesoft/upplz-plugins

# 2. 플러그인 설치 (설치 중 API Key 입력 프롬프트가 뜬다)
/plugin install upplz@upplz-tools

# 3. 완료 후 반영
/reload-plugins
```

- 설치 시 `plugin.json`의 `userConfig`에 따라 **Upplz API Key**를 물어 keychain에 안전 저장한다(`sensitive: true`).
- 저장된 값은 번들 MCP 서버의 `Authorization: Bearer ${user_config.api_key}` 헤더로 주입된다. 서버는 이 키를 해시 조회해 `customerId`를 확정한다(`src/mcp/auth.ts`).
- **API 키 발급**: 고객은 upplz 대시보드(설정 URL / `create_upload_link`)에서 본인 키를 먼저 발급받아야 한다.

## 버전 업데이트 배포

1. 본 저장소 `plugin/`에서 변경 → 리뷰/머지.
2. `plugin.json`의 `version` 상향.
3. 전용 저장소 `upplz-plugins`의 `plugins/upplz/`에 동기화 + `marketplace.json`의 `version` 반영 → push.
4. 고객은 `/plugin marketplace update upplz-tools` 후 재설치/갱신.

> `userConfig`는 **설치 시점에만** 프롬프트된다. 업데이트만으로는 재입력되지 않으므로, 키 변경이 필요하면 재설치를 안내한다.

## ⚠️ MCP 엔드포인트 설정

`plugin.json`의 `api_endpoint` 기본값은 플레이스홀더(`https://YOUR-UPPLZ-ENDPOINT`)다 — **실제 엔드포인트 주소는 공개 저장소에 박지 않는다**(개인/내부 인프라 노출 방지). 실제 주소는 별도(로컬 메모리·온보딩 채널)로 관리한다.

- 고객은 설치 시 `userConfig` 프롬프트에서, upplz 온보딩(설정 URL/대시보드)으로 안내받은 **실제 엔드포인트**를 입력한다.
- 정식/대규모 배포용 **안정적 프로덕션 엔드포인트**(고정 도메인·이중화)가 정해지면 기본값으로 교체할 수 있다. 개인 머신 주소는 공개 기본값으로 두지 않는다.
- Streamable-HTTP는 헤드리스/대화형 인증(MFA·패스프레이즈 프롬프트) 불가 — 인증은 Bearer 토큰(API 키) 단일 경로를 유지한다.

## 개발/내부 테스트 (배포 아님)

정식 배포 채널과 별개로, 개발 중에는 마켓플레이스 없이 로컬 로드한다:

```bash
claude --plugin-dir /경로/upplz.git/plugin
/reload-plugins   # 스킬 수정 즉시 반영
```

> 주의: `--plugin-dir` 로드는 `userConfig` 프롬프트를 띄우지 않으므로 번들 MCP 서버의 `api_key`가 비어 인증이 안 될 수 있다. 스킬(오케스트레이터) 자체를 점검할 때 쓰고, MCP 도구까지 포함한 전체 흐름은 `/plugin install`(키 프롬프트 포함)로 검증한다. 이미 별도로 upplz MCP 서버를 설정해 둔 개발 환경이면 번들 서버와 이름이 겹칠 수 있으니 주의.
