# 0003. Compose profiles for local-llm and observability

Date: 2026-08-10
Status: Accepted — defined in `compose.yaml`, not yet exercised (both profiles off)

## Context
Ollama/exo-style local LLM serving and a Grafana/Loki/Prometheus stack are
both valuable but not needed for the core loop to run, and both cost
meaningful RAM on a Mini — Ollama alone needs 8–24GB for a useful model.

## Decision
Both live in `compose.yaml` from day one as Compose **profiles**
(`local-llm`, `observability`), off by default, so adopting either is a flag
(`--profile local-llm`) rather than a redesign.

## Alternatives considered
- **Separate compose files per concern** — rejected: more files to keep in
  sync; profiles achieve the same opt-in behavior in one file.
- **Add them only when needed** — rejected: cheap to stub now, and having the
  service definition already reviewed removes a "does this even fit" question
  later.

## Consequences
The default footprint stays ~1.5GB (core six containers only). Turning on
exo-as-provider doesn't need this profile at all (exo runs outside Docker on
Ambar's Mac cluster, reached via `CCE_OPENAI_BASE_URL`) — the `local-llm`
profile is specifically for running Ollama *inside* this Docker network,
which remains optional.
