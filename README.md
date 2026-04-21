# tailscale-funnel-manager

JSON config based helper for launching a local app and managing `tailscale funnel` open/close/status flows.

## Goals

- keep per-app config in JSON
- optionally keep one dummy anchor funnel open so the machine DNS stays warm
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

## Port convention

Use app ports starting from `8000` by default.

Recommended rule of thumb:
- `8000+` for app funnels
- keep the anchor funnel separate, typically on `80`
- when onboarding a new app, prefer moving the local dev server to an `8000` range port instead of picking an arbitrary `8081`, `5173`, or similar port

This keeps URLs predictable and makes multi-app Funnel usage easier to reason about.

## Config shape

See `schemas/funnel-app.schema.json` and `examples/mental-health-app.json`.

The `funnel.anchor` block keeps a dummy funnel open before the app-specific funnel is opened. This mirrors the old `~/funnel.sh` behavior where one background funnel stayed alive to preserve DNS availability. The default recommendation is to keep both `background` and `https` enabled so multiple funnels can coexist more safely.

## Roadmap

- inject env values into app launch
- add HTTP healthcheck probing
- support hostname-aware funnel config
- make anchor funnel strategy configurable per machine or per app
