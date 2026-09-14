# Phase 5 — Judge Worker evolution

> Consolidated from the existing Phase 5 walkthrough. This is a project-level synthesis; detailed technical notes should be split into focused files as the project continues.

## Problem

Run untrusted submission code for a testcase while controlling timeout, stdout/stderr volume, memory pressure, cancellation, process lifecycle, and result classification.

## Evolution

1. Run one external command.
2. Add timeout.
3. Drain output concurrently so a full pipe cannot block the process.
4. Enforce an output limit to protect the worker.
5. Coordinate main execution and reader threads through explicit ownership and synchronization.
6. Record lifecycle events before interpreting an ambiguous exit code.
7. Add inner and outer timeouts as defense in depth.
8. Treat Docker cleanup as a separate resource-ownership concern.
9. Scale from one testcase to compile-plus-testcase aggregation with an explicit fail-fast policy.

## Main lessons

- A process runner is a lifecycle coordinator, not a single `run()` call.
- Timeout and output draining are coupled: killing the process does not replace draining its pipes.
- Exit code alone is insufficient after forced termination; preserve the event/cause first.
- Concurrency becomes manageable when ownership, visibility, ordering, and atomicity are explicit.
- Cleanup deserves its own policy and verification path.

## Trade-offs

- Fail-fast saves resources and latency but may hide later testcase failures.
- Keeping bounded output improves safety but loses part of the diagnostic stream.
- Multiple timeout layers improve containment but increase coordination complexity.
- Docker isolation improves safety boundaries but adds startup and cleanup cost.

## Source

See `E:\Learning\LeetCodeFake-Phase5-Judge-Worker-Evolution-Walkthrough.md` for the detailed step-by-step walkthrough and `E:\Learning\LeetCodeFake-Phase5-Judge-Worker-Mindmap.md` for the original map.

