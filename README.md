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

Use app ports starting from `8000` by default for local services.

Recommended rule of thumb:
- local app ports can freely use `8000+`
- public Funnel exposure is constrained by Tailscale rules and is best treated separately from local app ports
- for the simplest public exposure, prefer app `https: false` so Funnel binds the public root URL on 443 to the chosen local port
- keep the anchor funnel separate when needed for DNS stability

This keeps local development predictable while acknowledging that public Funnel ports follow different rules.

## Config shape

See `schemas/funnel-app.schema.json` and `examples/mental-health-app.json`.

The `funnel.anchor` block keeps a dummy funnel open before the app-specific funnel is opened. This mirrors the old `~/funnel.sh` behavior where one background funnel stayed alive to preserve DNS availability.

In practice, there is an important distinction:
- local app ports are flexible
- public Funnel exposure has port constraints from Tailscale
- setting app `funnel.https` to `false` can be the most practical way to expose one app publicly on the root Funnel URL

## Roadmap

- inject env values into app launch
- add HTTP healthcheck probing
- support hostname-aware funnel config
- make anchor funnel strategy configurable per machine or per app
