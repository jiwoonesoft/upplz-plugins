# upplz-plugins

웹 게임/앱을 iOS(맥 로컬 빌드)·Android(맥·윈도 로컬 Docker 하네스)로 빌드·배포하는 **upplz** Claude Code 플러그인 마켓플레이스.

## 설치

```bash
/plugin marketplace add jiwoonesoft/upplz-plugins
/plugin install upplz@upplz-tools
# 설치 중 Upplz API Key 입력(→ keychain 저장) 후:
/reload-plugins
# 배포 설정 시작(프로젝트당 1회):
/upplz:init
```

설치 한 번으로 **스킬 5개 + 커맨드 `/upplz:init` + 원격 upplz MCP 서버 연결 + API 키**가 함께 구성됩니다.

- **스킬**: `upplz-upload-build-only`(테스트 업로드)·`upplz-upload`(출시 업로드)·`upplz-metadata`·`upplz-icon`·`upplz-screenshot`.
- **커맨드**: `/upplz:init` — 배포 설정 원스톱(내장 `/init` 이 아니다).
- **API 키**: upplz 대시보드에서 **프로젝트를 만든 뒤 프로젝트 페이지에서** 발급받아 설치 시 입력(키 1개 = 프로젝트 1개).
- 상세: [플러그인 README](./plugins/upplz/README.md) · [배포 가이드](./DISTRIBUTION.md).

## 저장소 구조

```
.claude-plugin/marketplace.json   # 마켓플레이스 카탈로그
plugins/upplz/                    # 플러그인 (upplz.git 의 plugin/ 동기화본)
DISTRIBUTION.md                   # 배포자용 가이드
```

플러그인 소스 원본은 upplz 서버 저장소의 `plugin/` 이며, 릴리스마다 `plugins/upplz/`로 동기화하고 `plugin.json`·`marketplace.json`의 `version`을 올립니다.
