# 0001. Modular monolith: one image, three process entrypoints

Date: 2026-08-10
Status: Accepted — implemented in Phase 0

## Context
The four architectural layers (context store, data products, agent runtime,
governance) need to run somewhere. A single-node Mac Mini with one operator
rules out a service-mesh approach, but the API, background jobs, and (later)
agent sessions have genuinely different failure and load profiles — a wedged
agent session or a heavy embedding job shouldn't take the API down with it.

## Decision
One Python codebase, one Docker image, three process entrypoints (`api`,
`worker`, `agent-runtime`) distinguished only by start command. Module
boundaries inside the codebase (`cce.store`, `cce.products`, `cce.agents`,
`cce.governance`, `cce.llm`) mirror the layers and are the future service
boundaries if ever split out.

## Alternatives considered
- **True microservices per layer** — rejected: multiplies build time, version
  skew, and inter-service auth for zero benefit at POC scale.
- **Single process, no separation** — rejected: couples API availability to
  worker/agent health.

## Consequences
Cheap to run on a Mini (~1.5GB RAM for the core), simple deploys, and a clear
path to splitting a process into its own deployable in cloud later without a
rewrite — only if a measured need appears (see ADR 0011 for the same
deferral pattern applied to the graph layer).
