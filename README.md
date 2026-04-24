# tailscale-funnel-manager

**머신당 config 1개.** 그 안에 앱 여러 개를 나열하고, `bin/funnel` 하나로 한 번에 기동/종료/상태확인.

## 운영 원칙

- 한 머신에는 `examples/config.json` 같은 **단일 config 파일 하나**만 둡니다.
- 그 머신에서 Funnel로 열어야 하는 **모든 앱**을 이 파일의 `apps[]` 안에 나열합니다.
- 앱을 추가/제거하는 것은 이 파일 하나를 수정하는 것과 같습니다. 여러 config를 돌려쓰지 않습니다.
- 결과: "이 머신에서 뭐가 떠있어야 하지?" 에 대한 답이 항상 **config 1개**에 들어 있음.

## Commands

운영 명령은 전부 이 config 하나를 인자로 받습니다.

```bash
bin/funnel up     examples/config.json
bin/funnel down   examples/config.json
bin/funnel status examples/config.json
```

- `up`: config의 `apps[]`를 순서대로 기동하고 각 앱에 대해 `tailscale funnel`을 오픈. `funnel up` 은 각 앱을 독립적으로 기동하며, 일부 앱이 실패해도 나머지는 계속 기동한다. 마지막에 성공/실패 요약을 출력한다.
- `down`: 같은 config를 기반으로 `tailscale funnel reset` 후 `apps[]`를 역순으로 정리.
- `status`: 같은 config의 포트별 listen 상태 + `tailscale funnel status`.

> 다른 config를 쓰지 않는 이유: "지금 이 머신에서 뭐가 떠있는지"의 정답을 한 파일로 고정하기 위함입니다.

## Config shape

최상위 키는 2개뿐: `anchor`(선택), `apps`(필수, 배열).

```json
{
  "anchor": {
    "enabled": true,
    "port": 80,
    "https": true,
    "background": true
  },
  "apps": [
    {
      "appName": "mental-health-app",
      "port": 8000,
      "cwd": "/Users/atflow/repos/mental-health-app",
      "command": "PORT=8000 node server.js",
      "startupWaitSeconds": 20,
      "healthcheckPath": "/",
      "logPath": "/tmp/mental-health-app.log",
      "pidFile": "/tmp/mental-health-app.pid",
      "funnel": { "https": false, "background": true }
    },
    {
      "appName": "kms-django",
      "port": 10000,
      "cwd": "/Users/atflow/repos/KMS/django",
      "command": "bash -lc 'source .venv/bin/activate && python manage.py runserver 0.0.0.0:10000'",
      "startupWaitSeconds": 20,
      "healthcheckPath": "/health/",
      "logPath": "/tmp/kms-django.log",
      "pidFile": "/tmp/kms-django.pid",
      "funnel": { "https": true, "background": true }
    },
    {
      "appName": "kms-react",
      "port": 8443,
      "cwd": "/Users/atflow/repos/KMS/react",
      "command": "pnpm run dev",
      "startupWaitSeconds": 30,
      "healthcheckPath": "/",
      "logPath": "/tmp/kms-react.log",
      "pidFile": "/tmp/kms-react.pid",
      "funnel": { "https": true, "background": true }
    }
  ]
}
```

이게 `examples/config.json`이며, **이 저장소가 가진 유일한 표준 예시**입니다. 머신마다 자기 환경에 맞게 이 파일을 한 벌 복사/수정해서 쓰면 됩니다.

### 동작 요약

- `up`:
  1. `anchor.enabled=true`면 config당 1회 anchor funnel을 엽니다.
  2. `apps[]`를 순서대로 순회하며
     - `cwd`에서 `command`를 background로 기동 (`logPath`에 append, PID는 `pidFile`에 기록)
     - 해당 앱의 `port`가 listen 상태가 될 때까지 `startupWaitSeconds`만큼 대기
     - `tailscale funnel`을 해당 앱의 `port`/`funnel.https`/`funnel.background` 설정대로 오픈
  3. 마지막에 `tailscale funnel status` 1회 출력
  4. 앱별 실패 처리:
     - 포트 준비 실패 → 해당 앱 프로세스 정리 후 다음 앱으로 진행 (`PORT_TIMEOUT`)
     - `tailscale funnel` 오픈 실패 → 앱 프로세스는 유지하고 다음 앱으로 진행 (`FUNNEL_FAILED`, 로컬에서 포트로 접근 가능)
  5. 루프 종료 후 성공/실패 요약 출력. 성공이 1개라도 있으면 exit 0, 전부 실패할 때만 exit 1.
- `down`:
  1. `tailscale funnel reset` 1회
  2. `apps[]`를 **역순**으로 pidFile 기반 정리
  3. `tailscale funnel status` 출력
- `status`:
  1. 각 앱의 `port`에 대해 `lsof -nP -iTCP:<port> -sTCP:LISTEN`
  2. `tailscale funnel status`

### 필드 설명

| 위치 | 키 | 설명 |
| --- | --- | --- |
| top | `anchor.enabled` | config당 1회 기동할 dummy funnel 사용 여부 (기본 false) |
| top | `anchor.port` / `https` / `background` | anchor funnel의 포트 / https / bg 플래그 |
| app | `appName` | 로그/PID 기본 경로에 사용됨 (**필수**) |
| app | `port` | 로컬에서 앱이 listen할 포트 (**필수**) |
| app | `cwd` | 프로세스 기동 디렉토리 |
| app | `command` | shell로 실행할 기동 명령. 공백/따옴표 포함 가능 |
| app | `startupWaitSeconds` | 포트 오픈까지 대기 초 (기본 15) |
| app | `healthcheckPath` / `healthcheckUrl` | 문서/메모용 (현재는 참고 필드) |
| app | `logPath` | 기본값 `/tmp/funnel-<appName>.log` |
| app | `pidFile` | 기본값 `/tmp/funnel-<appName>.pid` |
| app | `funnel.https` | `tailscale funnel --https`로 열지 여부 |
| app | `funnel.background` | `tailscale funnel --bg` 여부 |
| app | `env` | (예약) 프로세스 환경변수 주입용 |

스키마 원본은 `schemas/funnel-app.schema.json` 참고. `additionalProperties=false`로 엄격하게 잡혀있어 오타는 즉시 드러납니다.

## Port convention

- 로컬 앱 포트는 `8000+`에서 자유롭게 사용 (예: 8000, 8443, 10000).
- 공용 Funnel 노출은 Tailscale 규칙에 의존하므로 로컬 포트와 분리해서 생각하세요.
  - 가장 간단한 공용 노출: 해당 앱의 `funnel.https`를 `false`로 두고 Tailscale이 루트 URL(443)을 로컬 포트로 바인딩.
- DNS 워밍을 위해 anchor funnel을 하나 따로 유지할 수 있습니다 (`anchor.enabled=true`).

## Example

이 저장소는 **예시 파일을 하나만** 둡니다.

- `examples/config.json` — `mental-health-app` + `kms-django` + `kms-react`를 한 config로 관리하는 표준 예시.

머신에 뭘 추가로 띄우고 싶으면, 이 파일의 `apps[]`에 항목을 더하는 식으로 확장하세요. 다른 `*.json`을 늘리지 않는 것이 이 프로젝트의 운영 약속입니다.

## Roadmap

- `env` 값 실제 주입 (현 스키마 예약)
- HTTP healthcheck 프로빙 (`healthcheckPath` / `healthcheckUrl`)
- hostname-aware funnel config
- anchor funnel 전략의 머신/앱 단위 커스터마이즈
