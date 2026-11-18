# 0021. Partners get a compiled distribution image

Date: 2026-10-12
Status: **Accepted — implemented** (`Dockerfile.dist`, `tools/build_dist.py`, `.github/workflows/dist.yml`, `python -m cce <command>`)

## Context

EazeWork's team wants to run CCE themselves: deploy it, connect hosted models, compare
them. The source image (ADR-0005) ships every module as readable Python, which a handful
of employees could paste into an AI tool and rebuild from. We are not hiding the design,
but we don't want to hand over the code as text.

## Decision

A second image, `ghcr.io/<owner>/cce-dist`, built per recipient by a manual workflow:

- Application modules and scripts are compiled to native extensions with Cython
  (`annotation_typing=False`, `binding=True`, so pydantic, SQLAlchemy and FastAPI behave as
  in the source build); module/class/function docstrings in the app are stripped first.
- No `.py` source remains except package `__init__` files, the `python -m cce` dispatcher
  and two-line script launchers. The Dockerfile fails the build if any is left.
- Readable by necessity: migrations, the HRMS pack YAML (including the question sets, kept
  because they are EazeWork's own questions and the way to compare models), the demo page,
  and string constants such as prompts inside the compiled files.
- `CCE_BUILD_ID` (e.g. `eazework-2026-10`) is baked in and reported by `/healthz`, so a
  copy can be traced to its recipient.
- linux/amd64 only (partners run Linux cloud servers). Published only after the compiled
  image passes the walking skeleton, the EazeWork checks and a docs ingest through the
  deploy bundle.
- Compiled modules can't be run with `python -m cce.x`, so operator commands go through
  `python -m cce <command>`, which works in both images.

## Consequences

- Copying the code into an AI tool no longer works; a determined engineer can still study
  behaviour and extract strings. The evaluation agreement (no decompiling, no AI tools,
  return at the end) carries the rest.
- A bug seen only in the compiled build is debugged by reproducing it on the source build;
  each dist build re-runs the end-to-end gate, so differences surface before publishing.
- The source image stays private to the owner.
