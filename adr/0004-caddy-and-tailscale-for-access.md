# 0004. Caddy for proxy/TLS; Tailscale for private access

Date: 2026-08-10
Status: Accepted — Caddy defined in `compose.prod.yaml`; Tailscale is a host-level step, not yet performed

## Context
The Mini needs to be reachable (by Ambar, and later collaborators) without
either running with zero access control or standing up public-internet
exposure for a POC.

## Decision
Caddy as the only container with published ports (reverse proxy + automatic
TLS). Tailscale installed on the Mini host (not containerized) for private
network access — `https://cce.<tailnet>.ts.net` reachable with zero public
ports open by default.

## Alternatives considered
- **nginx/traefik** — rejected: more config to hand-write for the same
  zero-config-TLS outcome Caddy gives for free.
- **Public exposure from day one** — rejected: no reason to accept that
  attack surface before there's a second real user.

## Consequences
Public exposure becomes a deliberate later decision, not a default. Egress/
ingress list stays small and auditable — useful when this maps to cloud
security groups later (Deployment doc §11).
