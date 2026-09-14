Dưới đây là bản **evolution walkthrough hoàn chỉnh** của Lab 3. Mình giữ đúng cách bạn đã học: bắt đầu từ phiên bản đơn giản nhất, mỗi step chỉ nâng cấp khi **bug của phiên bản hiện tại buộc ta phải nâng cấp**.

Mục tiêu cuối cùng không phải chỉ là nhớ một đoạn code Judge Worker, mà là nhìn được:

```text
simple implementation
        ↓
bug / failure mode xuất hiện
        ↓
hiểu WHY
        ↓
thêm đúng mechanism để giải quyết
        ↓
mechanism mới lại tạo ra vấn đề mới
        ↓
tiếp tục evolve
```

---

# Big picture — chúng ta đã evolve qua những gì?

```text
Step 0
readAllBytes()
    ↓
BUG: block trước timeout

Step 1
waitFor(timeout) trước
    ↓
BUG: stdout pipe đầy → child block

Step 2
Reader Thread đọc song song
    ↓
BUG: nếu gọi run() thay start() thì vẫn block Main

Step 3
readerThread.start()
    ↓
Main monitor + Reader drain concurrent
    ↓
BUG: outputBuffer tăng vô hạn → host OOM

Step 4
Chunk reading + MAX_OUTPUT
    ↓
cần shared state outputExceeded
    ↓
BUG: local boolean không mutate được trong lambda
    + concurrency visibility

Step 5
AtomicBoolean
    ↓
Reader signal OUTPUT_LIMIT
    ↓
Main timeout riêng
    ↓
cần coordinate lifecycle

Step 6
process.waitFor() + readerThread.join()
    ↓
BUG: join() có thể block vô hạn

Step 7
bounded join()
    ↓
BUG: timeout và output-limit có thể race
    ↓
2 boolean có thể cùng true

Step 8
AtomicReference<TerminationReason>
CAS
    ↓
first cause wins

Step 9
Main owns final verdict
    ↓
reason + exitCode + output

Step 10
Production hardening
    ↓
Docker CLI != container
inner timeout + outer timeout
explicit container cleanup
```

---

# Step 0 — Phiên bản đơn giản nhất

Giả sử ta muốn:

1. chạy process
2. lấy output
3. xem process có timeout không
4. evaluate kết quả

Ta rất dễ viết:

```java
Process process = pb.start();

String output = new String(
        process.getInputStream().readAllBytes(),
        StandardCharsets.UTF_8
);

boolean done =
        process.waitFor(3, TimeUnit.SECONDS);

if (!done) {
    process.destroyForcibly();
    return;
}

int exitCode = process.exitValue();
```

Nhìn code rất hợp lý:

```text
start
 ↓
read output
 ↓
wait
 ↓
check result
```

Nhưng có bug cực lớn.

---

## Problem 0 — `readAllBytes()` có thể block vô hạn

`readAllBytes()` conceptually:

```text
read
read
read
read
...
until EOF
```

EOF chỉ tới khi stream kết thúc.

Giả sử user code:

```java
while (true) {
}
```

không terminate.

Process vẫn alive.

stdout vẫn chưa đóng.

Do đó:

```java
process.getInputStream().readAllBytes();
```

có thể chờ mãi.

Flow thật:

```text
Main
 │
 ├─ process.start()
 │
 ├─ readAllBytes()
 │       │
 │       └── BLOCK...
 │
 X
waitFor(3 seconds)
```

Dòng timeout thậm chí **không được chạy tới**.

### Learning note

> Timeout chỉ hữu ích nếu execution flow thực sự tới được đoạn code kiểm tra timeout.

---

# Step 1 — Đưa `waitFor(timeout)` lên trước

Ta thử sửa:

### Before

```java
String output =
        new String(process.getInputStream().readAllBytes());

boolean done =
        process.waitFor(3, TimeUnit.SECONDS);
```

### After

```java
boolean done =
        process.waitFor(3, TimeUnit.SECONDS);

if (!done) {
    process.destroyForcibly();
    return;
}

String output =
        new String(
                process.getInputStream().readAllBytes(),
                StandardCharsets.UTF_8
        );
```

Mental model:

```text
start process
     ↓
wait max 3s
     ↓
process done?
     ↓
read output
```

Giờ timeout đã chạy được.

Nhưng design lại có một failure mode mới.

---

# Problem 1 — stdout pipe có finite capacity

Process không write output trực tiếp vào Java String.

Có một pipe giữa process và Judge:

```text
Child Process
     │
     │ write stdout
     ▼
┌───────────────┐
│   OS pipe     │
│ finite buffer │
└───────────────┘
     │
     │ read
     ▼
Judge
```

Giả sử user:

```java
while (true) {
    System.out.println("AAAAAAAAAAAAAAAAAAAA");
}
```

Main đang:

```java
process.waitFor(3, TimeUnit.SECONDS);
```

và chưa đọc stdout.

Pipe dần:

```text
0%
20%
50%
80%
100%
```

Khi pipe đầy:

```text
Child:
write(...)
   │
   └── BLOCK
```

Process chưa terminate vì đang block khi write.

Main:

```text
waitFor()
```

đang chờ process terminate.

Ta có:

```text
Main
waitFor process
    │
    │
    ▼
Process
wait for stdout space
    │
    │
    ▼
space chỉ xuất hiện nếu Main read
```

Đây là deadlock-like workflow.

Sau 3 giây Main có thể báo timeout dù nguyên nhân thực tế là Judge **không drain stdout**.

### Learning note

> Khi làm việc với child process, stdout/stderr không phải “data có sẵn để đọc sau”. Chúng là streams cần được drain trong lúc process đang chạy.

---

# Step 2 — Tạo Reader Thread

Ta cần:

```text
Main Thread
→ monitor lifecycle / timeout

Reader Thread
→ drain stdout continuously
```

Architecture:

```text
                  Process
                     │
                  stdout
                     │
                     ▼
                Reader Thread
                     │
                read chunks


Main Thread
    │
    └── waitFor(timeout)
```

Hai việc chạy concurrent.

Code đầu tiên:

```java
ByteArrayOutputStream outputBuffer =
        new ByteArrayOutputStream();

Thread readerThread = new Thread(() -> {
    try {
        InputStream input =
                process.getInputStream();

        byte[] chunk = new byte[8192];

        while (true) {
            int n = input.read(chunk);

            if (n == -1) {
                break;
            }

            outputBuffer.write(chunk, 0, n);
        }

    } catch (IOException e) {
        e.printStackTrace();
    }
});
```

Nhưng tạo `Thread` object chưa làm nó chạy.

---

# Problem 2 — `run()` vs `start()`

Nếu viết:

```java
readerThread.run();
```

thì không có thread mới.

Nó giống:

```java
someMethod();
```

Execution:

```text
Main Thread
   │
   ├─ readerThread.run()
   │        │
   │        ├─ input.read()
   │        ├─ input.read()
   │        └─ ...
   │
   X
process.waitFor()
```

Nếu `input.read()` block thì Main lại block.

Ta quay về bug cũ.

---

# Step 3 — dùng `start()`

Phải:

```java
readerThread.start();
```

JVM bắt đầu một execution thread mới và thread đó chạy `run()`.

```text
Main Thread                     Reader Thread

readerThread.start()
      │
      └───────────────────────► run()
                                  │
                                  ├─ read()
                                  ├─ read()
                                  └─ read()

process.waitFor(3s)
      │
      └─ monitor process
```

Code:

```java
ByteArrayOutputStream outputBuffer =
        new ByteArrayOutputStream();

Thread readerThread = new Thread(() -> {
    try {
        InputStream input =
                process.getInputStream();

        byte[] chunk = new byte[8192];

        int n;

        while ((n = input.read(chunk)) != -1) {
            outputBuffer.write(chunk, 0, n);
        }

    } catch (IOException e) {
        e.printStackTrace();
    }
});

readerThread.start();

boolean done =
        process.waitFor(3, TimeUnit.SECONDS);

if (!done) {
    process.destroyForcibly();
}

readerThread.join();
```

Đây là bước tiến lớn:

```text
Reader drains output
        +
Main monitors timeout
```

### Learning note

> `run()` = chạy code trên current thread.
> `start()` = bắt đầu thread execution mới, thread mới chạy `run()`.

---

# Problem 3 — Output buffer có thể ăn hết RAM của Judge

Ta đang có:

```java
ByteArrayOutputStream outputBuffer =
        new ByteArrayOutputStream();
```

Reader:

```java
while (...) {
    outputBuffer.write(...);
}
```

User có thể:

```java
while (true) {
    System.out.println("AAAAAAAA...");
}
```

Output:

```text
1 MB
10 MB
100 MB
500 MB
1 GB
...
```

`ByteArrayOutputStream` tiếp tục grow.

Điểm cực kỳ quan trọng:

```text
Docker container memory limit
        ≠
Judge JVM memory limit
```

Giả sử:

```text
Container RAM limit = 128 MB
```

User process không cần giữ 1 GB trong RAM.

Nó chỉ:

```text
generate 8KB
write
discard
generate tiếp
```

Trong khi host:

```text
Judge JVM
outputBuffer
1GB
2GB
...
→ OOM
```

### Learning note

> Resource limit của sandbox không tự bảo vệ memory mà Judge dùng để consume sandbox output.

---

# Step 4 — Chunk reading + Output Limit

Ta đặt:

```java
private static final long MAX_OUTPUT_BYTES =
        1024 * 1024; // 1 MB
```

Reader cần count **bytes đã đọc**.

Bạn ban đầu từng viết conceptually:

```java
totalBytes += 1;
```

nhưng đó là đếm số lần `read()`.

Nếu:

```java
int n = input.read(chunk);
```

trả:

```text
n = 8192
```

nghĩa là nhận được 8192 bytes.

Phải:

```java
totalBytes += n;
```

Code:

```java
byte[] chunk = new byte[8192];

long totalBytes = 0;

while (true) {

    int n = input.read(chunk);

    if (n == -1) {
        break;
    }

    totalBytes += n;

    if (totalBytes > MAX_OUTPUT_BYTES) {
        // output limit exceeded
        break;
    }

    outputBuffer.write(chunk, 0, n);
}
```

---

## Tại sao không dùng `readAllBytes()` rồi check?

Sai:

```java
byte[] output =
        input.readAllBytes();

if (output.length > MAX_OUTPUT_BYTES) {
    ...
}
```

Vấn đề là bạn kiểm tra **sau khi đã đọc toàn bộ**.

Nếu process output 10GB:

```text
read 10GB
    ↓
OOM
    ↓
không bao giờ tới:
if (length > 1MB)
```

Chunked read cho phép:

```text
read 8KB
check

read 8KB
check

...

cross 1MB
STOP
```

### Learning note

> Khi data có thể lớn, không biết trước size, hoặc được sinh theo thời gian → streaming/chunk processing.

---

# Problem 4 — Reader phát hiện exceed, Main cần biết

Reader detect:

```text
output > 1MB
```

Main sau cùng cần quyết định:

```text
OUTPUT_LIMIT
```

Ta cần communication:

```text
Reader
   │
   └── WRITE outputExceeded

Main
   │
   └── READ outputExceeded
```

Đây là shared mutable state.

---

# Shared mutable state checkpoint

Trong Lab:

```text
MAX_OUTPUT
→ shared: yes
→ mutable: no

timedOut
→ shared: no
→ mutable: yes

totalBytes
→ shared: no
→ mutable: yes

outputExceeded
→ shared: yes
→ mutable: yes

outputBuffer
→ shared: yes
→ mutable: yes
```

`outputBuffer` đặc biệt:

Reader write trong lúc chạy.

Main chỉ read **sau `join()`**.

```text
Reader
write write write
        │
        ▼
terminate
        │
        ▼
join returns
        │
        ▼
Main read
```

Nên access được ordered/coordinated.

---

# Problem 5 — plain local boolean trong lambda

Ta muốn:

```java
boolean outputExceeded = false;

Thread reader = new Thread(() -> {
    outputExceeded = true;
});
```

Java không compile.

Captured local variable phải `final` hoặc effectively final.

Reader đang cố **reassign** local variable.

---

# Step 5 — `AtomicBoolean`

Ta thay bằng object:

```java
AtomicBoolean outputExceeded =
        new AtomicBoolean(false);
```

Reader:

```java
outputExceeded.set(true);
```

Main:

```java
outputExceeded.get();
```

Điểm quan trọng:

```text
outputExceeded local reference
        │
        ▼
┌────────────────────┐
│   AtomicBoolean    │
│                    │
│ false → true       │
└────────────────────┘
```

Local reference không bị reassign.

Chỉ state trong object thay đổi.

Do đó lambda vẫn capture được reference.

Code:

```java
AtomicBoolean outputExceeded =
        new AtomicBoolean(false);

ByteArrayOutputStream outputBuffer =
        new ByteArrayOutputStream();

Thread readerThread = new Thread(() -> {
    try {
        InputStream input =
                process.getInputStream();

        byte[] chunk = new byte[8192];

        long totalBytes = 0;

        while (true) {
            int n = input.read(chunk);

            if (n == -1) {
                break;
            }

            totalBytes += n;

            if (totalBytes > MAX_OUTPUT_BYTES) {
                outputExceeded.set(true);

                process.destroyForcibly();

                break;
            }

            outputBuffer.write(chunk, 0, n);
        }

    } catch (IOException e) {
        e.printStackTrace();
    }
});
```

---

# `volatile` vs `AtomicBoolean`

Nếu đây là field:

```java
private volatile boolean outputExceeded;
```

và chỉ có pattern:

```text
Reader: set true
Main:   read
```

thì `volatile` có thể đủ về visibility.

`volatile` đảm bảo communication semantics phù hợp:

```text
Reader WRITE true
      │
      ▼
Main subsequent READ
      │
      ▼
sees true
```

Nhưng `volatile` không giải quyết operation kiểu:

```java
counter++;
```

vì:

```text
READ
 ↓
MODIFY
 ↓
WRITE
```

không atomic.

Hai thread:

```text
counter = 0

Thread A             Thread B

READ 0               READ 0
+1                   +1
WRITE 1              WRITE 1

expected 2
actual   1
```

`AtomicInteger.incrementAndGet()` hoặc lock mới giải quyết read-modify-write race đó.

### Learning note

> `volatile` chủ yếu giải quyết visibility/communication.
> `AtomicXxx` còn cung cấp atomic operations như CAS.

---

# Step 6 — Main monitor timeout

Reader đã xử lý output.

Main xử lý execution time.

```java
readerThread.start();

boolean done =
        process.waitFor(
                3,
                TimeUnit.SECONDS
        );
```

Cực kỳ quan trọng:

```text
process.waitFor()
→ WAIT FOR PROCESS

readerThread.join()
→ WAIT FOR THREAD
```

`done == true` chỉ có nghĩa:

> process đã terminate trong khoảng Main chờ.

Không có nghĩa:

```text
AC
exit 0
Reader finished
output correct
```

---

## Scenario

Process:

```text
running
   ↓
exit
```

Main:

```text
waitFor() → true
```

Reader vẫn có thể:

```text
draining final bytes
doing cleanup
not terminated yet
```

Process lifecycle và Reader lifecycle khác nhau.

---

## Timeout case

```java
boolean timedOut = false;

if (!done) {
    timedOut = true;
    process.destroyForcibly();
}
```

`false` nghĩa là process **vẫn alive khi timeout window hết**.

Không được:

```java
if (!done) {
    int exitCode = process.exitValue(); // sai logic
}
```

vì chưa chắc process đã terminate.

Flow:

```text
waitFor(3s)
     │
     └─ false
          │
          ▼
      timedOut = true
          │
          ▼
   destroyForcibly()
```

---

# Tại sao kill process giúp Reader thoát `read()`?

Reader có thể đang:

```java
input.read(chunk);
```

Process:

```java
while (true) {
    // không output
}
```

Pipe:

```text
không data
nhưng producer vẫn alive
```

Reader:

```text
read()
→ BLOCK
```

Main timeout:

```text
kill process
```

Process kết thúc.

stdout side đóng.

Reader thường nhận:

```text
EOF
```

hoặc I/O failure.

Rồi thoát.

```text
Process killed
     ↓
stdout closes
     ↓
read() released
     ↓
Reader exits
```

---

# Step 7 — `join()`

Sau process termination/kill:

```java
readerThread.join();
```

Main nói:

> Chờ Reader terminate rồi tôi mới inspect final output.

Code:

```java
readerThread.start();

boolean done =
        process.waitFor(
                3,
                TimeUnit.SECONDS
        );

boolean timedOut = false;

if (!done) {
    timedOut = true;
    process.destroyForcibly();
}

readerThread.join();

String output =
        outputBuffer.toString(
                StandardCharsets.UTF_8
        );
```

`join()` còn có memory-ordering significance.

Reader actions trước terminate become visible phù hợp cho thread successfully `join()`.

Do đó Main đọc `outputBuffer` sau join là design hợp lý.

---

# Problem 6 — `join()` có thể block vô hạn

Ta có timeout cho:

```java
process.waitFor(3s)
```

nhưng:

```java
readerThread.join();
```

không có timeout.

Hai timeout hoàn toàn khác nhau.

```text
process.waitFor(3s)
→ bounded

readerThread.join()
→ potentially unbounded
```

Nếu Reader vì bug hoặc resource issue không terminate:

```text
Main
 │
 └─ join()
      │
      └── forever
```

Timeout của process không magically áp dụng cho Thread.

---

# Step 8 — bounded join

Ta nâng cấp:

```java
readerThread.join(1000);
```

Sau đó:

```java
if (readerThread.isAlive()) {
    // Reader chưa terminate
}
```

Quan trọng:

```text
isAlive() == true
```

chỉ cho biết:

> Thread chưa terminate.

Không có nghĩa chắc chắn:

```text
infinite loop
```

Nó có thể:

```text
blocked on I/O
waiting for lock
slow cleanup
bug
infinite loop
```

Đây là cùng mental model với exit code:

```text
WHAT ≠ WHY
```

Code:

```java
readerThread.join(1000);

if (readerThread.isAlive()) {
    // infrastructure / cleanup problem
}
```

Production hơn một chút có thể close stream:

```java
InputStream stdout =
        process.getInputStream();
```

Reader dùng chính `stdout`.

Nếu Reader không exit:

```java
stdout.close();

readerThread.join(250);
```

Vì interrupt không nhất thiết unblock `InputStream.read()` đáng tin cậy; close resource thường trực tiếp hơn.

---

# Problem 7 — Hai boolean có thể race

Hiện tại:

```java
AtomicBoolean outputExceeded =
        new AtomicBoolean(false);

boolean timedOut = false;
```

Timeline:

```text
2.999s
Reader detects output limit

3.000s
Main timeout
```

Có thể cuối cùng:

```text
outputExceeded = true
timedOut       = true
```

Main nhìn:

```text
true
true
```

thì cause nào thắng?

Bạn từng nghĩ:

> dùng AtomicBoolean error, chỉ cho một thread update cause.

Đúng hướng, nhưng boolean không lưu được **cause nào**.

---

# Step 9 — AtomicReference + TerminationReason

Ta chuyển từ nhiều flag sang state machine.

Ban đầu:

```java
enum TerminationReason {
    RUNNING,
    OUTPUT_LIMIT,
    TIMEOUT
}
```

```java
AtomicReference<TerminationReason> reason =
        new AtomicReference<>(
                TerminationReason.RUNNING
        );
```

Reader:

```java
reason.compareAndSet(
        TerminationReason.RUNNING,
        TerminationReason.OUTPUT_LIMIT
);
```

Main:

```java
reason.compareAndSet(
        TerminationReason.RUNNING,
        TerminationReason.TIMEOUT
);
```

CAS:

```text
COMPARE
   +
SET
```

được thực hiện atomically.

---

## Race example

Reader:

```text
CAS
RUNNING → OUTPUT_LIMIT

SUCCESS
```

Main một chút sau:

```text
CAS
RUNNING → TIMEOUT

FAIL
```

Final:

```text
OUTPUT_LIMIT
```

Ngược lại:

```text
Main wins first
→ TIMEOUT
```

State machine:

```text
          OUTPUT_LIMIT
         /
RUNNING
         \
          TIMEOUT
```

Không còn state:

```text
outputExceeded = true
timedOut = true
```

---

## Naming refinement

Sau này có thể đổi `RUNNING` thành:

```java
NONE
```

vì state này thực chất có nghĩa:

> Chưa ghi nhận forced termination cause.

```java
enum TerminationReason {
    NONE,
    OUTPUT_LIMIT,
    OUTER_TIMEOUT
}
```

Tên này chính xác hơn khi process đã natural exit nhưng chưa có forced termination reason.

---

# Step 10 — Main vẫn là final verdict authority

Reader được quyền:

```text
detect OUTPUT_LIMIT
record cause
kill fail-fast
```

Nhưng Reader không return verdict cuối.

Main cuối cùng đọc execution facts.

```text
Reader
   │
   ├─ detect event
   ├─ record cause
   └─ kill


Main
   │
   ├─ wait lifecycle
   ├─ coordinate cleanup
   ├─ inspect reason
   ├─ inspect exitCode
   ├─ inspect output
   └─ FINAL VERDICT
```

Điểm cực kỳ quan trọng:

> WHO detects an event ≠ WHO owns final decision.

---

# Exit code không phải cause

Ví dụ:

```text
exitCode = 137
```

có thể vì:

```text
OOM
external SIGKILL
output-limit kill
timeout kill
container cleanup
```

Nên:

```text
exitCode
→ termination result / symptom

terminationReason
→ execution context / cause

verdict
→ Judge classification
```

Main nên ưu tiên known context.

Ví dụ:

```java
TerminationReason finalReason =
        reason.get();

if (finalReason == TerminationReason.OUTPUT_LIMIT) {
    return OUTPUT_LIMIT;
}

if (finalReason == TerminationReason.OUTER_TIMEOUT) {
    return TLE;
}

int exitCode =
        process.exitValue();

if (exitCode != 0) {
    return RUNTIME_ERROR;
}

if (!output.equals(expected)) {
    return WRONG_ANSWER;
}

return ACCEPTED;
```

---

# Evolution của Reader — từ sai đến hoàn chỉnh

Phiên bản ban đầu bạn từng thử:

```java
Thread readerThread = new Thread(() -> {

    try {
        InputStream input =
                process.getInputStream();

        byte[] chunk =
                new byte[8192];

        long totalBytes = 0;

        while (totalBytes < Byte(input)) {

            int n =
                    input.read(chunk);

            if (input.EOF) {
                break;
            }

            totalBytes += 1;

            if (totalBytes > MAX_OUTPUT_BYTES) {
                outputExceeded = true;
                process.destroyForcibly();
                break;
            }

            outputBuffer.write(
                    chunk,
                    0,
                    n
            );
        }

    } catch (IOException e) {
        e.printStackTrace();
    }
});
```

Các vấn đề:

```text
Byte(input)
→ stream không nhất thiết biết total size

input.EOF
→ không có API như vậy

totalBytes += 1
→ đếm số read calls
→ không đếm bytes

outputExceeded = true
→ local captured variable problem
→ concurrency semantics

Thread object
→ chưa start
```

Sau evolution:

```java
AtomicReference<TerminationReason> reason =
        new AtomicReference<>(
                TerminationReason.NONE
        );

ByteArrayOutputStream outputBuffer =
        new ByteArrayOutputStream();

InputStream stdout =
        process.getInputStream();

Thread readerThread =
        new Thread(() -> {

    try {
        byte[] chunk =
                new byte[8192];

        long totalBytes = 0;

        while (true) {

            int n =
                    stdout.read(chunk);

            if (n == -1) {
                break;
            }

            totalBytes += n;

            if (totalBytes > MAX_OUTPUT_BYTES) {

                reason.compareAndSet(
                        TerminationReason.NONE,
                        TerminationReason.OUTPUT_LIMIT
                );

                if (process.isAlive()) {
                    process.destroyForcibly();
                }

                break;
            }

            outputBuffer.write(
                    chunk,
                    0,
                    n
            );
        }

    } catch (IOException e) {
        // xử lý ở version production phía dưới
    }
});
```

---

# Full Lab 3 — phiên bản consolidated

Đây là phiên bản gom toàn bộ những gì đã học.

```java
import java.io.ByteArrayOutputStream;
import java.io.IOException;
import java.io.InputStream;
import java.nio.charset.StandardCharsets;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.atomic.AtomicReference;

public class JudgeLab3 {

    private static final long MAX_OUTPUT_BYTES =
            1024 * 1024; // 1 MB

    private static final long PROCESS_TIMEOUT_MS =
            3000;

    private static final long READER_JOIN_TIMEOUT_MS =
            1000;

    enum TerminationReason {
        NONE,
        OUTPUT_LIMIT,
        OUTER_TIMEOUT
    }

    enum Verdict {
        ACCEPTED,
        WRONG_ANSWER,
        RUNTIME_ERROR,
        TIME_LIMIT_EXCEEDED,
        OUTPUT_LIMIT_EXCEEDED,
        JUDGE_ERROR
    }

    record RunResult(
            Verdict verdict,
            int exitCode,
            String output,
            long durationMs,
            TerminationReason terminationReason
    ) {
    }

    public static RunResult run(
            ProcessBuilder pb,
            String expectedOutput
    ) throws Exception {

        pb.redirectErrorStream(true);

        long startedAt =
                System.nanoTime();

        Process process =
                pb.start();

        InputStream stdout =
                process.getInputStream();

        AtomicReference<TerminationReason> reason =
                new AtomicReference<>(
                        TerminationReason.NONE
                );

        AtomicReference<Throwable> readerError =
                new AtomicReference<>();

        ByteArrayOutputStream outputBuffer =
                new ByteArrayOutputStream();

        Thread readerThread =
                new Thread(
                        () -> {
                            try {
                                byte[] chunk =
                                        new byte[8192];

                                long totalBytes = 0;

                                while (true) {

                                    int n =
                                            stdout.read(chunk);

                                    if (n == -1) {
                                        break;
                                    }

                                    totalBytes += n;

                                    if (
                                            totalBytes
                                                    > MAX_OUTPUT_BYTES
                                    ) {

                                        reason.compareAndSet(
                                                TerminationReason.NONE,
                                                TerminationReason.OUTPUT_LIMIT
                                        );

                                        if (process.isAlive()) {
                                            process.destroyForcibly();
                                        }

                                        break;
                                    }

                                    outputBuffer.write(
                                            chunk,
                                            0,
                                            n
                                    );
                                }

                            } catch (IOException e) {

                                /*
                                 * Nếu process bị kill có chủ đích,
                                 * stream đóng và read có thể fail.
                                 *
                                 * Khi đó IOException không nhất thiết
                                 * là Judge infrastructure error.
                                 */
                                if (
                                        reason.get()
                                                == TerminationReason.NONE
                                ) {
                                    readerError.compareAndSet(
                                            null,
                                            e
                                    );
                                }
                            }
                        },
                        "judge-output-reader"
                );

        /*
         * Không dùng readerThread.run().
         *
         * start() tạo execution thread mới.
         */
        readerThread.start();

        /*
         * Main waits for PROCESS,
         * không phải Reader Thread.
         */
        boolean processDone =
                process.waitFor(
                        PROCESS_TIMEOUT_MS,
                        TimeUnit.MILLISECONDS
                );

        if (!processDone) {

            /*
             * First cause wins.
             */
            reason.compareAndSet(
                    TerminationReason.NONE,
                    TerminationReason.OUTER_TIMEOUT
            );

            /*
             * Dù CAS thắng hay thua,
             * process vẫn phải được cleanup.
             */
            if (process.isAlive()) {
                process.destroyForcibly();
            }

            /*
             * destroyForcibly() không nên được xem
             * như "process đã chết synchronously".
             *
             * Cho nó một bounded wait nữa.
             */
            process.waitFor(
                    500,
                    TimeUnit.MILLISECONDS
            );
        }

        /*
         * Main waits for THREAD.
         *
         * Không dùng join() vô hạn.
         */
        readerThread.join(
                READER_JOIN_TIMEOUT_MS
        );

        if (readerThread.isAlive()) {

            /*
             * Thử giải phóng blocking read.
             */
            try {
                stdout.close();
            } catch (IOException ignored) {
            }

            readerThread.interrupt();

            readerThread.join(250);
        }

        long durationMs =
                TimeUnit.NANOSECONDS.toMillis(
                        System.nanoTime()
                                - startedAt
                );

        /*
         * Reader vẫn không terminate:
         * Judge infrastructure problem.
         */
        if (readerThread.isAlive()) {

            return new RunResult(
                    Verdict.JUDGE_ERROR,
                    -1,
                    "",
                    durationMs,
                    reason.get()
            );
        }

        /*
         * Reader I/O failure không giải thích được
         * bởi intentional termination.
         */
        if (readerError.get() != null) {

            return new RunResult(
                    Verdict.JUDGE_ERROR,
                    process.isAlive()
                            ? -1
                            : process.exitValue(),
                    "",
                    durationMs,
                    reason.get()
            );
        }

        String output =
                outputBuffer.toString(
                        StandardCharsets.UTF_8
                );

        TerminationReason finalReason =
                reason.get();

        /*
         * Known cause có priority
         * hơn exit code.
         */
        if (
                finalReason
                        == TerminationReason.OUTPUT_LIMIT
        ) {

            return new RunResult(
                    Verdict.OUTPUT_LIMIT_EXCEEDED,
                    process.isAlive()
                            ? -1
                            : process.exitValue(),
                    output,
                    durationMs,
                    finalReason
            );
        }

        if (
                finalReason
                        == TerminationReason.OUTER_TIMEOUT
        ) {

            return new RunResult(
                    Verdict.TIME_LIMIT_EXCEEDED,
                    process.isAlive()
                            ? -1
                            : process.exitValue(),
                    output,
                    durationMs,
                    finalReason
            );
        }

        /*
         * Không có forced termination cause,
         * nhưng process vẫn alive:
         * infrastructure inconsistency.
         */
        if (process.isAlive()) {

            return new RunResult(
                    Verdict.JUDGE_ERROR,
                    -1,
                    output,
                    durationMs,
                    finalReason
            );
        }

        int exitCode =
                process.exitValue();

        if (exitCode != 0) {

            return new RunResult(
                    Verdict.RUNTIME_ERROR,
                    exitCode,
                    output,
                    durationMs,
                    finalReason
            );
        }

        if (!output.equals(expectedOutput)) {

            return new RunResult(
                    Verdict.WRONG_ANSWER,
                    exitCode,
                    output,
                    durationMs,
                    finalReason
            );
        }

        return new RunResult(
                Verdict.ACCEPTED,
                exitCode,
                output,
                durationMs,
                finalReason
        );
    }
}
```

---

# Execution flow của final Lab 3

```text
Main Thread
    │
    ├─ pb.start()
    │
    ├─ create Reader
    │
    ├─ reader.start()
    │        │
    │        └────────────────────────────┐
    │                                     │
    ├─ process.waitFor(timeout)            │
    │                                     │
    │                                     ▼
    │                              Reader Thread
    │                                     │
    │                              stdout.read()
    │                                     │
    │                              totalBytes += n
    │                                     │
    │                      ┌──────────────┴────────────┐
    │                      │                           │
    │                 under limit                 over limit
    │                      │                           │
    │                buffer.write()            CAS reason
    │                                                  │
    │                                           OUTPUT_LIMIT
    │                                                  │
    │                                            kill process
    │
    ├─ waitFor returns?
    │
    ├─ false
    │    │
    │    ├─ CAS NONE → OUTER_TIMEOUT
    │    └─ kill process
    │
    ├─ bounded join Reader
    │
    ├─ inspect reader error
    │
    ├─ inspect termination reason
    │
    ├─ inspect exit code
    │
    ├─ compare output
    │
    ▼
FINAL VERDICT
```

---

# Test case evolution

Bây giờ Lab 3 nên được test bằng nhiều loại user program.

## Case 1 — Normal output

```java
public class Main {
    public static void main(String[] args) {
        System.out.print("42");
    }
}
```

Expected:

```text
process exits
Reader gets EOF
reason = NONE
exitCode = 0
output = "42"
expected = "42"

→ ACCEPTED
```

---

## Case 2 — Wrong answer

```java
public class Main {
    public static void main(String[] args) {
        System.out.print("41");
    }
}
```

```text
reason = NONE
exit = 0
output != expected

→ WRONG_ANSWER
```

Important:

```text
exit 0 ≠ Accepted
```

---

## Case 3 — Runtime error

```java
public class Main {
    public static void main(String[] args) {
        throw new RuntimeException();
    }
}
```

Nếu stderr merge stdout:

```java
pb.redirectErrorStream(true);
```

Reader vẫn drain stacktrace.

Final:

```text
reason = NONE
exit != 0

→ RUNTIME_ERROR
```

---

## Case 4 — Infinite loop, no output

```java
public class Main {
    public static void main(String[] args) {
        while (true) {
        }
    }
}
```

Reader:

```text
input.read()
→ block
```

Main:

```text
waitFor(3s)
→ false

CAS NONE → OUTER_TIMEOUT
kill
```

Reader được giải phóng.

Final:

```text
→ TLE
```

---

## Case 5 — Infinite output

```java
public class Main {
    public static void main(String[] args) {
        while (true) {
            System.out.println(
                    "AAAAAAAAAAAAAAAAAAAA"
            );
        }
    }
}
```

Reader:

```text
read
count
read
count
...
> 1MB
```

Then:

```text
CAS NONE → OUTPUT_LIMIT
kill
```

Main waitFor có thể return:

```text
true
```

vì process đã bị Reader kill.

Nhưng:

```text
done == true
```

không được hiểu thành success.

Main thấy:

```text
reason = OUTPUT_LIMIT
```

Final:

```text
OUTPUT_LIMIT_EXCEEDED
```

---

# Step 11 — Docker-specific evolution

Cho đến đây code xử lý Java `Process`.

Nhưng command thật của Judge là kiểu:

```java
ProcessBuilder pb =
        new ProcessBuilder(
                "docker",
                "run",
                "--rm",
                "--memory=128m",
                "--cpus=1",
                "--network=none",
                "-v",
                workDir + ":/app",
                "judge-java",
                "bash",
                "-c",
                "cd /app && java Main"
        );
```

Process Java nhận được từ:

```java
Process process = pb.start();
```

đại diện trực tiếp cho:

```text
docker CLI process
```

không phải trực tiếp:

```text
java Main inside container
```

Process tree conceptually:

```text
Spring Boot JVM
     │
     │ ProcessBuilder
     ▼
docker CLI
     │
     ▼
Docker daemon
     │
     ▼
Container
     │
     ▼
bash
     │
     ▼
java Main
```

Do đó:

```java
process.destroyForcibly();
```

direct target là docker CLI process.

Không nên assume:

```text
kill docker CLI
=
container chắc chắn đã được cleanup
```

---

# Step 12 — Explicit container identity

Production Judge nên biết container nào đang chạy.

Ví dụ:

```java
String containerName =
        "judge-" + UUID.randomUUID();
```

Command:

```java
ProcessBuilder pb =
        new ProcessBuilder(
                "docker",
                "run",
                "--rm",
                "--name",
                containerName,
                "--memory=128m",
                "--cpus=1",
                "--network=none",
                "-v",
                workDir.toAbsolutePath()
                        + ":/app",
                "judge-java",
                "bash",
                "-c",
                "cd /app && java Main"
        );
```

Khi timeout/output-limit:

```text
1. kill docker CLI handle
2. explicitly force-remove container
```

Helper:

```java
private static void forceCleanupContainer(
        Process dockerProcess,
        String containerName
) {

    if (dockerProcess.isAlive()) {
        dockerProcess.destroyForcibly();
    }

    try {
        Process cleanup =
                new ProcessBuilder(
                        "docker",
                        "rm",
                        "-f",
                        containerName
                )
                .redirectErrorStream(true)
                .start();

        cleanup.waitFor(
                2,
                TimeUnit.SECONDS
        );

    } catch (Exception e) {
        // log infrastructure cleanup failure
    }
}
```

Điều này nâng architecture từ:

```text
I killed my direct child process
```

sang:

```text
I explicitly cleaned up the sandbox resource
```

---

# Step 13 — Inner timeout + Outer timeout

Một production judge thường không chỉ có một timeout.

Ví dụ trong container:

```bash
timeout 2s java Main
```

Outer Judge:

```java
process.waitFor(
        5,
        TimeUnit.SECONDS
);
```

Architecture:

```text
USER TIME LIMIT
      │
      ▼
inner timeout
inside sandbox
~2s


JUDGE SAFETY LIMIT
      │
      ▼
outer timeout
host side
~5s
```

Inner timeout:

```text
enforce contestant execution limit
```

Outer timeout:

```text
protect Judge if:
docker hangs
shell hangs
cleanup hangs
inner timeout fails
unexpected infrastructure issue
```

Đây là defense in depth.

### Learning note

> Inner timeout bảo vệ correctness của testcase limit. Outer timeout bảo vệ Judge infrastructure.

---

# Một distinction cuối cùng rất quan trọng

Qua toàn bộ evolution này, có ba layer khác nhau:

```text
EVENT
↓
"output exceeded"
"outer timeout occurred"


PROCESS FACT
↓
done
exitCode
isAlive


VERDICT
↓
OUTPUT_LIMIT
TLE
RE
WA
AC
```

Không nên merge chúng.

Ví dụ:

```text
exitCode = 137
```

chỉ là fact.

```text
reason = OUTPUT_LIMIT
```

là context.

```text
verdict = OUTPUT_LIMIT_EXCEEDED
```

là classification.

---

# Toàn bộ knowledge map bạn vừa học

```text
ProcessBuilder
│
├─ start OS process
│
├─ Process handle
│   ├─ waitFor()
│   ├─ isAlive()
│   ├─ exitValue()
│   └─ destroyForcibly()
│
├─ stdout / stderr
│   └─ InputStream
│       ├─ read()
│       ├─ EOF = -1
│       └─ pipe backpressure
│
├─ Reader Thread
│   ├─ Thread object
│   ├─ run()
│   ├─ start()
│   └─ join()
│
├─ Streaming
│   ├─ chunk[]
│   ├─ n = read(chunk)
│   ├─ totalBytes += n
│   └─ ByteArrayOutputStream
│
├─ Resource protection
│   ├─ timeout
│   └─ output limit
│
├─ Concurrency
│   ├─ shared state
│   ├─ mutable state
│   ├─ visibility
│   ├─ ordering
│   └─ atomicity
│
├─ Synchronization
│   ├─ volatile
│   ├─ AtomicBoolean
│   ├─ AtomicReference
│   ├─ CAS
│   └─ join / happens-before
│
├─ State ownership
│   ├─ Reader detects events
│   ├─ Main monitors lifecycle
│   └─ Main owns final verdict
│
└─ Judge semantics
    ├─ event
    ├─ cause
    ├─ exit status
    └─ verdict
```

Nếu viết vào learning log, câu tổng kết quan trọng nhất của evolution này có thể là:

> **Tôi bắt đầu từ một `ProcessBuilder` rất đơn giản với `readAllBytes()` và `waitFor(timeout)`, nhưng khi mô phỏng các failure mode thực tế, tôi nhận ra process execution là bài toán concurrent coordination: stdout phải được drain song song để tránh pipe backpressure, output phải được bounded để bảo vệ host, timeout và output-limit có thể race nên termination cause phải được ghi nhận atomically, và final verdict phải được quyết định từ execution context chứ không chỉ từ exit code.**

Đó thực sự là bước chuyển từ **“biết dùng ProcessBuilder”** sang **“biết thiết kế một process supervisor/Judge execution engine”**.


