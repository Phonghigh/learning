---
name: capture-learning
description: Convert raw notes into a structured project learning record and classify its dominant archive category.
---

# Capture learning

Use when the user pastes raw learning, asks to log a lesson, or says `capture learning`.

## Output

Create one Markdown note under `project/<project>/learning/` with:

1. Title and metadata: date, project, phase, dominant category, status.
2. The problem and why it mattered.
3. Context and constraints.
4. Approaches considered or attempted.
5. Chosen approach and implementation shape.
6. Evidence and verification.
7. Trade-offs and rejected alternatives.
8. What I learned.
9. Reusable principles.
10. LinkedIn angle candidates.

Update the project `INDEX.md` only if persistence is requested. If facts are missing, write `Unknown` or `Not verified`; do not fill gaps from assumptions.

