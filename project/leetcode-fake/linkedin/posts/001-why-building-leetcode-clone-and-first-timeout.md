# Part 001 — I am building a LeetCode Clone

Status: Draft - saved in LinkedIn article editor

- Project phase: Phase 5 — Judge Worker
- Current version: Version 1 → Version 2
- Previous post: None
- Next post: Part 002 — Version 3: stdout pipes and concurrent draining

I am building my own LeetCode Clone.

Not just to copy the UI. I want to understand what happens after we click **Submit**:

- How does Spring manage a submission?
- How does the OS create and manage a process?
- How do threads work with a process?
- What does a container really protect?
- How does a Judge decide AC, WA, RE, or TLE?

This project has already taught me many useful things: ProcessBuilder, stdout pipes, thread synchronization, Docker, and more.

So I am sharing the journey as a series.

I will not start with a complete architecture. I will build it step by step:

~~~text
simple version
    ↓
failure mode
    ↓
understand the cause
    ↓
smallest useful improvement
    ↓
new failure mode
~~~

Right now, I am working on the **Judge Worker**.

A real Judge Worker must:

~~~text
compile untrusted code
  -> run it in a sandbox
  -> run many test cases
  -> limit time, memory, and output
  -> clean up resources
  -> classify the verdict
~~~

That is too much for a first step.

So I reduce the problem to one test case:

~~~text
input + command
  -> run an external process
  -> collect output and execution state
  -> decide one test-case verdict
~~~

## Version A — the first working version

The first question is simple:

How can Java run an external program, collect its output, and know how it ended?

### Code before

~~~java
ProcessBuilder pb =
        new ProcessBuilder("java", "Main");

Process process = pb.start();

String output = new String(
        process.getInputStream().readAllBytes(),
        StandardCharsets.UTF_8
);

int exitCode = process.waitFor();
~~~

This works for a small process that ends quickly and prints little output.

But there is an important detail:

~~~text
exit code 0 ≠ Accepted
~~~

It only means the command ended normally.

If the program prints 41 while the expected answer is 42:

~~~text
exit code = 0
actual output = 41
expected output = 42

verdict = WRONG_ANSWER
~~~

### The first failure

What if the child never stops?

~~~java
while (true) {
}
~~~

readAllBytes() may wait forever for EOF. The code never reaches waitFor().

~~~text
start
  -> readAllBytes()  [blocks]
  -> waitFor()       [never runs]
~~~

I need a timeout.

## Version A+ — add a timeout

### Code after

~~~java
ProcessBuilder pb =
        new ProcessBuilder("java", "Main");

Process process = pb.start();

boolean done =
        process.waitFor(
                3,
                TimeUnit.SECONDS
        );

if (!done) {
    process.destroyForcibly();
}

String output = new String(
        process.getInputStream().readAllBytes(),
        StandardCharsets.UTF_8
);

int exitCode = done
        ? process.exitValue()
        : -1;
~~~

Now the Worker waits for at most three seconds. If the process is still alive, it kills it.

waitFor(timeout) answers one lifecycle question:

~~~text
Did the process finish before the deadline?
~~~

It does not answer:

- Was the process successful?
- Was the answer correct?
- What caused the failure?

The next problem appears immediately:

> While Main is waiting with waitFor(), who is reading stdout and stderr?

An OS pipe has a limited buffer. If the user program prints continuously, the pipe can become full. Then the child blocks on write(), while the parent waits for the child to exit.

That can look like a false TLE.

In the next post, I will continue from this exact Version A+ and explain stdout pipes, backpressure, and why a Judge must drain output while the process is running.

**Lesson:** a timeout and a verdict are different things.

What failure mode would you check next in Version A+?

#Java #SpringBoot #Docker #BackendDevelopment #SystemDesign #BuildInPublic

## Handoff to next post

- Current implementation: Version 2 waits before draining output.
- New failure mode: stdout/stderr pipes have limited capacity.
- Next mechanism: concurrent output draining.
