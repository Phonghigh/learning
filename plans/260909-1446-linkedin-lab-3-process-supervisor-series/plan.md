# LinkedIn Series Plan: Building LeetCodeFake

## Direction

Document the real evolution of LeetCodeFake, a learning-oriented online judge.

Narrative:
problem -> smallest working solution -> failure mode -> investigation -> design evolution -> trade-off -> reusable lesson

Context:
- Current project focus: Phase 5, Judge Worker evolution.
- Learning method: record problem, rejected approaches, chosen design, trade-offs, evidence, lesson, and LinkedIn story.
- Core thesis: a Judge Worker is a lifecycle coordinator, evidence collector, and verdict engine, not merely a ProcessBuilder wrapper.

## Series positioning

Working title: Building LeetCodeFake — Từ một bài submit đến Online Judge

Audience: Java/backend developers, systems learners, students building portfolio projects, and engineers interested in sandboxing, Docker, concurrency, and execution systems.

Promise: Mỗi bài giải thích một quyết định kỹ thuật trong quá trình build LeetCodeFake: bắt đầu từ bản đơn giản nhất, rồi để failure mode buộc kiến trúc trưởng thành.

Cadence: 2 posts/week. Recommended first season: 12 weeks.

## Series roadmap

### Arc 1 — From request to accepted answer

1. Why build a LeetCode Clone?
2. Reduce the system to one testcase.
3. The first external-process runner.
4. Why exit code is not Accepted.
5. Compile once, run many testcases.
6. Turn execution into a domain result.
7. Arc recap.

### Arc 2 — Judge Worker evolution

8. readAllBytes() can defeat your timeout.
9. stdout pipe backpressure and false TLE.
10. run() vs start(): the fake concurrency bug.
11. Draining output can still cause host OOM.
12. Output limits and bounded collection.
13. Ordering, visibility, and atomicity.
14. Process lifecycle vs reader lifecycle.
15. Why join() also needs a timeout.
16. Race between timeout and output limit.
17. Event, cause, exit code, and verdict.
18. Arc recap: ProcessBuilder became a process supervisor.

### Arc 3 — Sandbox and submission lifecycle

19. Docker CLI is not the container.
20. Inner timeout vs outer timeout.
21. Explicit container cleanup.
22. Verdict taxonomy: CE, RE, TLE, WA, AC.
23. Fail-fast vs complete testcase diagnostics.
24. Aggregating testcase results into a submission result.
25. Production-hardening recap.

Arc 2 can be the launch point because it is the strongest current material, but the overall series should preserve the product journey.

## Post briefs

### 1. Why build a LeetCode Clone?

Hook: Tôi không xây LeetCodeFake vì muốn clone giao diện LeetCode. Tôi xây nó để hiểu điều gì xảy ra sau nút Submit.

Cover submission -> compile -> sandbox -> testcases -> verdict. Explain why an online judge combines application code with OS, process, runtime, and resource-control concerns.

Lesson: the project is a learning vehicle for systems engineering.

### 2. Thu nhỏ bài toán về một testcase

Hook: Nếu bắt đầu bằng compile, Docker, queue và nhiều testcase cùng lúc, bạn rất dễ học không được gì cả.

Reduce the system to input + command -> output + execution state -> testcase verdict. Explain what is intentionally postponed.

Lesson: reduce the problem until the next failure is understandable.

### 3. ProcessBuilder: bản đầu tiên chạy được

Hook: Bản đầu tiên của Judge Worker có thể chỉ là vài dòng Java.

Cover command configuration, process start, stdout, wait, and exit code. Explain the parent/child stream viewpoint.

Lesson: a minimal implementation exposes the next problem.

### 4. exit code 0 không phải Accepted

Hook: Chương trình in sai đáp án nhưng vẫn exit code 0. Judge phải làm gì?

Separate command success from answer correctness. Show actual 41, expected 42, exit 0 -> Wrong Answer.

Lesson: OS status and product status are different models.

### 5. Compile once, run many testcases

Hook: Vì sao compile không nên nằm trong vòng lặp testcase?

Explain the submission-level compilation gate, compile-once flow, testcase result, submission result, and fail-fast policy.

Lesson: lifecycle boundaries simplify aggregation.

### 6. Turn raw execution into evidence

Hook: Một RunResult gồm exitCode và output chưa đủ để giải thích vì sao submission fail.

Introduce duration, output bytes, timeout evidence, reader failure, termination reason, and captured output.

Lesson: observability is part of correctness.

### 7. readAllBytes() có thể vô hiệu hóa timeout

Hook: Bạn có thể viết waitFor(3 seconds) mà timeout vẫn không bao giờ chạy.

Explain EOF, infinite-loop child, and why control flow never reaches the timeout check.

Lesson: blocking I/O can invalidate lifecycle control.

### 8. stdout pipe tạo ra false TLE

Hook: User code chạy 500ms nhưng Judge báo TLE sau 3s. Không phải lúc nào TLE cũng do CPU.

Explain finite pipe capacity, child blocking on write, parent waiting, and concurrent stream draining.

Lesson: timeout monitoring and output draining solve different problems.

### 9. run() vs start(): concurrency giả

Hook: Tạo một Thread object không có nghĩa là bạn đã có concurrency.

Compare run() with start(), show Main monitor vs Reader drain, and mention stderr handling.

Lesson: the API call determines whether the architecture actually runs concurrently.

### 10. Drain output nhưng đừng làm host OOM

Hook: Container có memory limit nhưng Judge JVM vẫn có thể chết vì giữ stdout vô hạn.

Explain unbounded ByteArrayOutputStream, separate container and host memory, byte counting, and MAX_OUTPUT_BYTES.

Lesson: every untrusted stream needs a bounded collection policy.

### 11. Concurrency: ordering, visibility, atomicity

Hook: outputExceeded = true nghe đơn giản, nhưng có ba câu hỏi concurrency khác nhau phía sau.

Explain timing/order, visibility, atomicity, volatile, AtomicBoolean, AtomicInteger, Future/result transfer, and ownership.

Lesson: choose a primitive based on the actual problem.

### 12. Process lifecycle vs Reader lifecycle

Hook: Process đã exit không có nghĩa Reader đã đọc xong output.

Explain waitFor(), join(), bounded cleanup, and why Main awaits reader completion before final evaluation.

Lesson: every lifecycle needs its own deadline and owner.

### 13. Race giữa timeout và output limit

Hook: Hai boolean cùng true thì nguyên nhân nào thắng?

Show near-simultaneous race, replace independent flags with TerminationReason, and introduce AtomicReference/CAS or documented precedence.

Lesson: mutually exclusive causes should be modeled as one state.

### 14. Event, cause, exit code, verdict

Hook: Exit code 137 không tự nói đây là OOM, TLE hay Judge kill.

Separate event, process fact, cause, and verdict. Main owns the deterministic final verdict.

Lesson: do not classify from exit code alone.

### 15. Docker CLI is not the container

Hook: destroyForcibly() trên Docker CLI chưa chắc đã cleanup xong sandbox.

Explain the host JVM -> Docker CLI -> daemon -> container -> shell -> Java chain, direct-child limitations, explicit container identity, and cleanup.

Lesson: terminate and clean up the resource you actually own.

### 16. Phase 5 recap

Hook: Sau nhiều lần sửa bug, Judge Worker không còn là một method chạy process nữa.

Summarize external process, timeout, stream drain, output cap, coordination, cause, Docker cleanup, inner/outer timeout, and testcase aggregation.

Lesson: architecture is often discovered by failure modes.

## Standard post format

1. Project context: phase and current problem.
2. Hook.
3. Smallest implementation.
4. Failure mode.
5. Investigation.
6. One design evolution.
7. Trade-off.
8. Reusable lesson.
9. One technical question.
10. Footer: Building LeetCodeFake — Part X.

Keep one primary mechanism per post. Use code only when it clarifies the decision.

## Visual system

- Dark terminal/code aesthetic.
- Project phase badge.
- Red for failure, amber for risk, green for resolved behavior.
- Every visual shows old version -> failure -> new mechanism.
- Arc 1 uses system-boundary and result-model diagrams.
- Arc 2 uses pipe, thread, memory-flow, state, and evidence diagrams.
- Arc 3 uses Docker lifecycle and submission-aggregation diagrams.

## Publishing workflow

Before writing:
- Read the corresponding project learning file.
- Identify the exact implementation stage.
- Extract one rejected approach and why it failed.
- Select one code fragment or diagram.
- Define the one-sentence lesson.

Before publishing:
- Label planned behavior as planned.
- Do not claim production readiness from a conceptual walkthrough.
- Verify Java semantics and exit-code claims.
- Keep terminology consistent.
- Link to the previous post and project index when useful.

After publishing:
- Record recurring questions in the project learning log.
- Turn repeated confusion into follow-up posts.
- Measure saves and technical comments.
- Publish a phase recap after every 4–6 posts.

## Success criteria

- The series follows actual LeetCodeFake project phases.
- Every post has a concrete implementation or design decision.
- Rejected approaches and trade-offs are visible.
- Lab 3 is integrated as the Judge Worker arc, not isolated from the product.
- Final posts connect testcase execution to submission aggregation and Docker lifecycle.
- Claims are grounded in project learning files and verified experiments.

