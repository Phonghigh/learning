# Building LeetCodeFake — LinkedIn Series

Series goal: ghi lại quá trình tự xây một LeetCode Clone để học sâu hơn về Spring, OS, process, thread, container, concurrency và systems engineering.

## Reading order

| # | Title | Project version | Status |
|---|---|---|---|
| 001 | I am building a LeetCode Clone | Version 1 → Version 2 | Draft - saved in LinkedIn editor |
| 002 | Version 3: stdout pipes and concurrent draining | Version 2 → Version 3 | Planned |
| 003 | Reader thread: vừa drain output vừa monitor timeout | A++ | Planned |
| 004 | run() và start(): một lỗi concurrency rất dễ nhầm | A++ | Planned |
| 005 | Drain output vẫn có thể làm Judge JVM OOM | A+++ | Planned |
| 006 | Output limit và bounded collection | A+++ | Planned |
| 007 | Ordering, visibility và atomicity trong Judge Worker | A++++ | Planned |
| 008 | Process lifecycle khác reader lifecycle | A++++ | Planned |
| 009 | Timeout và output limit cùng xảy ra thì sao? | A+++++ | Planned |
| 010 | Từ exit code đến verdict: vì sao cần execution evidence | A+++++ | Planned |
| 011 | Docker CLI không phải container | Production | Planned |
| 012 | Phase recap: từ ProcessBuilder đến Judge Worker | Production | Planned |

## Context handoff rule

Mỗi bài mới phải đọc bài ngay trước đó, source learning tương ứng, code/version được ghi trong bài trước, và mục Next problem của bài trước.

Mỗi bài phải ghi lại:

- Previous post
- Current version
- What changed
- What this fixes
- Next problem

## Source context

- ../project-overview.md
- ../learning/phase-05-judge-worker-evolution.md
- ../../LeetCodeFake-Phase5-Judge-Worker-Evolution-Walkthrough.md
- ../../LeetCodeFake-Phase5-Judge-Worker-Mindmap.md
