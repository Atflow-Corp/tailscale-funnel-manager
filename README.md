# tailscale-funnel-manager

JSON config based helper for launching a local app and managing `tailscale funnel` open/close/status flows.

## Goals

- keep per-app config in JSON
- launch app process from config
- wait for local port readiness
- open `tailscale funnel`
- close and clean up consistently

## Commands

```bash
bin/funnel up examples/mental-health-app.json
bin/funnel down examples/mental-health-app.json
bin/funnel status examples/mental-health-app.json
```

## Config shape

See `schemas/funnel-app.schema.json` and `examples/mental-health-app.json`.

## Roadmap

- inject env values into app launch
- add HTTP healthcheck probing
- support hostname-aware funnel config
