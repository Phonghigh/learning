# Learning Archive Workspace

## Purpose

This workspace turns raw self-learning notes into durable project knowledge and LinkedIn-ready content.
The user may paste incomplete notes, chat transcripts, errors, design decisions, or code observations.

## Default workflow

When the user says `capture learning`, `log this`, or pastes a learning without another explicit task:

1. Identify the project. Prefer an existing project under `project/`; create a project folder only when the project is clearly new.
2. Classify the dominant technical category using the existing archive categories. Keep `project` for project-journey material, not for every technical lesson.
3. Extract: problem, context, attempted approaches, evidence, chosen approach, trade-offs, result, and reusable lesson.
4. Write the durable learning note under `project/<project>/learning/`.
5. Update the relevant project index and the root `INDEX.md` only when the user explicitly asks to persist the note.
6. Offer or create a mind map when the learning contains multiple decisions, phases, or dependencies.
7. Offer or create a LinkedIn draft only when requested, or when the user explicitly asks for content conversion.

## Classification rules

- `project`: project narrative, phase evolution, architecture journey, roadmap, milestone, or cross-cutting synthesis.
- `backend-api`: API contracts, service logic, backend data-shape decisions.
- `database`: schema, migration, constraint, query, persistence decisions.
- `devops-infra`: Docker, deployment, process lifecycle, environment, CI/CD, runtime operations.
- `frontend-ui`: UI state, rendering, component behavior, browser-side async flows.
- `gis-mapping`: maps, geospatial data, layers, projections, raster/vector behavior.
- Prefer one dominant category. Mention secondary topics in the note body instead of duplicating files.

## Writing rules

- Preserve uncertainty: label inferred causes and unverified assumptions.
- Explain trade-offs and rejected approaches; do not write a changelog.
- Separate facts observed from interpretation.
- Use Vietnamese when the input is Vietnamese; keep technical identifiers in English.
- Never invent metrics, test results, URLs, or implementation details.
- LinkedIn drafts must teach one clear idea, use a concrete story, avoid generic motivation, and end with a practical takeaway.

## File layout

```text
project/<project>/
  INDEX.md
  project-overview.md
  learning/*.md
  mindmaps/*.md
  linkedin/*.md
```

Use kebab-case file names. Do not create Markdown outside `E:\Learning\plans` or `E:\Learning\docs` unless the file is part of this existing archive structure.

## Available local skills

- `.Codex/skills/capture-learning/SKILL.md`
- `.Codex/skills/mindmap-learning/SKILL.md`
- `.Codex/skills/linkedin-post/SKILL.md`

