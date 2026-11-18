# 0019. Follow-up questions: the conversation rewrites the question, it is never a source

Date: 2026-10-10
Status: **Accepted — implemented** (`llm.standalone_question`, `/ask` `history`)

## Context

In the demo chat people ask follow-ups ("Can they be carried forward?", "What if I was 45
minutes late instead?"). Answered alone they fail or are refused. Feeding earlier answers
into the next answer would let a model repeat its own mistakes and make citations
meaningless.

## Decision

1. The client sends up to the last 3 turns as `history` with each `/ask`; nothing about
   the conversation is stored server-side.
2. If the question looks like a follow-up (short, or starts with "and / what about / what
   if…", or refers back with "it / they / that…"), the fast model rewrites it into one
   standalone question (no model: the previous question is attached as a heuristic).
3. Retrieval, eligibility, the question-kind and grounded gates, generation, the trace and
   any logged gap all use the **standalone question**. The answer is built only from the
   rules retrieved for it, never from earlier answers. Grounding takes the better of the
   original and rewritten phrasing.
4. `/ask` returns `understood_as`; the UI shows "Understood as: …" so a misreading is visible.
5. Changing who is asking (company or employee) starts a new conversation. Claims made in
   the chat ("I'm on probation") never override the profile.

## Consequences

- About one extra fast-model call (~1 s) on follow-ups only.
- Follow-up cases are part of the Widget question set (W-22…W-25).
