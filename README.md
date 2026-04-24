# tailscale-funnel-manager

JSON config 하나로 **여러 로컬 앱을 동시에 기동**하고, 각각에 대해 `tailscale funnel`을 열고/닫고 상태를 확인하는 헬퍼.

## Goals

- config 1개 = **여러 앱**
- 각 앱 프로세스 기동 + 포트 대기 + 개별 funnel 오픈을 순서대로 수행
- 필요하면 config 단위로 anchor funnel을 하나 유지해 DNS 워밍
- 종료/상태 확인도 동일 config로 일관되게

## Commands

```bash
bin/funnel up     examples/kms.json
bin/funnel down   examples/kms.json
bin/funnel status examples/kms.json
```

## Config shape

최상위 키는 2개뿐입니다: `anchor`(선택), `apps`(필수, 배열).

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

### 동작 요약

- `up`:
  1. `anchor.enabled=true`면 config당 1회 anchor funnel을 엽니다.
  2. `apps[]`를 순서대로 순회하며
     - `cwd`에서 `command`를 background로 기동 (`logPath`에 append, PID는 `pidFile`에 기록)
     - 해당 앱의 `port`가 listen 상태가 될 때까지 `startupWaitSeconds`만큼 대기
     - `tailscale funnel`을 해당 앱의 `port`/`funnel.https`/`funnel.background` 설정대로 오픈
  3. 마지막에 `tailscale funnel status` 1회 출력
  4. 어느 앱이 포트 준비에 실패하면: 에러 로그 + `tailscale funnel reset` + 이미 띄운 앱들을 best-effort로 정리한 뒤 exit 1. 세바님이 재실행하기 쉽게 설계.
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

## Examples

- `examples/mental-health.json` — 단일 앱
- `examples/kms.json` — kms-django + kms-react 2앱 동시 기동

## Roadmap

- `env` 값 실제 주입 (현 스키마 예약)
- HTTP healthcheck 프로빙 (`healthcheckPath` / `healthcheckUrl`)
- hostname-aware funnel config
- anchor funnel 전략의 머신/앱 단위 커스터마이즈
