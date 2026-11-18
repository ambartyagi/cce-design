# 0005. Registry-pulled, multi-arch, SHA-tagged images

Date: 2026-08-10
Status: Proposed — Phase 0 builds locally on-device; not yet pushed to a registry

## Context
The same image needs to run on arm64 (MacBook, Mini) and eventually amd64
(cloud). "What's actually deployed on the Mini?" needs a one-command answer,
and a debugged decision trace six months from now needs the exact image that
produced it to still be buildable.

## Decision
`docker buildx build --platform linux/arm64,linux/amd64`, pushed to a
registry (GHCR is the pragmatic default) even while the Mini is the only
consumer. Tags are the git SHA; a moving `latest` exists for dev only. `/healthz`
reports the running image's SHA.

## Alternatives considered
- **Build directly on the Mini** — what Phase 0 does today via `make prod-up`;
  fine as a bootstrap, but skips the reproducibility and pull-based-deploy
  properties this ADR wants as the standing state.

## Consequences
Makes MacBook→Mini deploys pull-based (`docker compose pull && up -d`) and
the cloud migration a re-homing rather than a rebuild. **Action item:** set
this up before Phase 3's Mac Mini move, not during it.
