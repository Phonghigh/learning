# LeetCodeFake Phase 5 — Judge Worker: giải quyết vấn đề từ nhỏ đến lớn

> Mục tiêu của tài liệu này không phải đưa một "đoạn Judge Worker hoàn chỉnh" để copy. Nó đi theo đúng đường phát triển của thiết kế: bắt đầu với bản nhỏ nhất có thể chạy, nhìn thấy lỗi mới mà bản đó sinh ra, rồi nâng cấp từng lớp `A -> A+ -> A++ -> ...`.
>
> Cách đọc: ở mỗi bước, hãy nắm **bản đang giải quyết điều gì**, **nó vẫn chưa giải quyết điều gì**, và **vì sao bước sau là cần thiết**. Đừng nhảy đến bản cuối trước khi hiểu lỗi của bản trước.

---

## 0. Bài toán cuối cùng và cách thu nhỏ nó

Judge Worker thực tế phải làm nhiều việc:

```text
Nhận submission
  -> compile code không tin cậy
  -> chạy code trong sandbox trên nhiều testcase
  -> giới hạn time / memory / CPU / network / output
  -> không để Worker bị treo hoặc OOM
  -> phân loại đúng CE / RE / TLE / WA / AC ...
  -> cleanup
  -> lưu và thông báo kết quả
```

Nếu bắt đầu thẳng từ toàn bộ danh sách này, ta sẽ bị ngợp. Vì vậy ta thu nhỏ bài toán về **một command cho một testcase**:

```text
input + command
  -> chạy external process
  -> nhận output + trạng thái execution
  -> quyết định verdict testcase
```

Khi một testcase đã an toàn và giải thích được, mới mở rộng lên compile và nhiều testcase.

**Note cần nhớ:** một Judge không chỉ chạy code. Nó là một hệ thống **quản lý lifecycle + thu thập evidence + ra quyết định** cho untrusted code.

---

## 1. Phiên bản A — chạy được một external command (happy path)

### Vấn đề 1

Làm thế nào để Java Worker chạy một chương trình bên ngoài, lấy output và biết chương trình kết thúc thế nào?

### A: bản nhỏ nhất

```java
ProcessBuilder pb = new ProcessBuilder("java", "Main");
Process process = pb.start();

String output = new String(process.getInputStream().readAllBytes());
int exitCode = process.waitFor();
```

### Bản này giải quyết được gì?

```text
Java Worker
  -> start external program
  -> đọc stdout
  -> chờ nó kết thúc
  -> lấy exit code
```

Với child nhỏ, kết thúc nhanh, in ít output, bản này đủ để làm Lab đầu tiên.

### Học được gì?

- `ProcessBuilder` là object **cấu hình command**.
- `pb.start()` tạo process bên ngoài và trả về Java `Process` handle.
- `Process` là handle Java; OS process mới là thứ thật sự đang chạy.
- `process.getInputStream()` đọc **stdout của child**. Tên `InputStream` là từ góc nhìn parent: output của child đi vào parent.
- `waitFor()` lấy/đợi lifecycle; exit code là kết quả termination của command.

### Điều rất dễ nhầm

```text
`exitCode == 0`
  != "đáp án đúng"

Nó chỉ gần nghĩa là
  "command này kết thúc bình thường theo convention của nó".
```

User program có thể in `41`, exit 0, trong khi expected là `42`. Verdict vẫn là `WRONG_ANSWER`.

### Vấn đề mới lộ ra

Nếu child chạy vô hạn, `readAllBytes()` có thể chờ mãi để nhận EOF. Code không bao giờ chạy đến `waitFor()`.

```text
start
  -> readAllBytes()  [chờ EOF]
  -> waitFor(...)    [không bao giờ tới]
```

Ta cần timeout, nên phải cải tiến.

---

## 2. Phiên bản A+ — thêm timeout để Worker không chờ vô hạn

### Vấn đề 2

Child có thể chạy vô tận. Worker phải có một giới hạn thời gian chờ và kill nó nếu quá hạn.

### A+: đưa `waitFor(timeout)` lên trước

```java
Process process = pb.start();

boolean done = process.waitFor(3, TimeUnit.SECONDS);
if (!done) {
    process.destroyForcibly();
}

String output = new String(process.getInputStream().readAllBytes());
```

### Bản này giải quyết được gì?

Với child không tạo nhiều output, Worker không bị treo vô hạn nữa:

```text
child vẫn sống sau 3 giây
  -> waitFor(...) trả false
  -> Worker kill child
```

### Học được gì?

`waitFor(timeout)` trả lời **một câu lifecycle**:

```text
Process đã kết thúc trước deadline chưa?
```

Không phải câu hỏi success/failure.

| Tình huống | `waitFor(3s)` | `exitValue()` |
|---|---:|---:|
| Child exit 0 ở 100ms | `true` | `0` |
| Child exit 7 ở 100ms | `true` | `7` |
| Reader/Judge kill child ở 400ms | `true` sau khi child terminate | thường non-zero |
| Child còn chạy ở 3s | `false` | chưa an toàn để gọi |

**Note cần nhớ:** timeout là **maximum wait**, không phải `sleep` đúng số giây. Nếu child bị kill ở 0.4s, `waitFor(2s)` có thể trả về ngay quanh 0.4s.

### Vấn đề mới lộ ra: pipe đầy

Bản A+ không đọc stdout/stderr khi đang chờ. Pipe OS có buffer hữu hạn:

```text
Child viết stdout
  -> pipe đầy
  -> write() của child block
  -> child không thể chạy tới exit
  -> parent waitFor(...) hết hạn
  -> false / kill
```

Điều này có thể tạo **TLE giả**:

```text
User computation vốn chỉ 500ms
nhưng in 100MB
parent không drain pipe
child bị block lúc print
parent thấy nó chưa exit ở giây 3
```

Vấn đề không phải user computation chậm; vấn đề là Judge làm child bị nghẹt I/O.

**Điều cần note:**

```text
Timeout monitor
  bảo vệ parent khỏi child sống quá lâu.

Drain output
  bảo vệ child khỏi block vì pipe đầy.
```

Hai nhiệm vụ khác nhau. Timeout không thay thế việc đọc pipe.

---

## 3. Phiên bản A++ — drain output đồng thời với monitor timeout

### Vấn đề 3

Ta cần vừa đọc output liên tục, vừa theo dõi timeout. Không thể làm tuần tự theo hai thứ tự đã thử:

```text
read trước -> timeout không chạy được
wait trước -> pipe có thể đầy
```

### A++: tách hai trách nhiệm chạy đồng thời

```text
Main / Judge task                 Output-reader task

start process                     đọc stdout/stderr liên tục
waitFor(outer deadline)           append output đã đọc
nếu cần thì kill                  kết thúc khi stream EOF
```

Pseudo-code ở mức ý tưởng:

```java
Process process = pb.start();

Future<CollectedOutput> reader = executor.submit(
    () -> drain(process.getInputStream())
);

boolean done = process.waitFor(outerTimeout, TimeUnit.SECONDS);
if (!done) {
    process.destroyForcibly();
}

CollectedOutput collected = reader.get();
```

Nếu không merge stderr bằng `redirectErrorStream(true)`, cần hai reader song song:

```text
stdout reader + stderr reader
```

Chỉ drain stdout mà bỏ stderr vẫn có thể làm child block trên stderr.

### Bản này giải quyết được gì?

- Timeout luôn được Main theo dõi.
- stdout/stderr được consume khi child đang chạy.
- Child không bị block chỉ vì Judge quên đọc pipe.
- Sau khi process kết thúc, reader nhận EOF và trả output đã thu thập.

### Học được gì?

- `readAllBytes()` không "xấu"; nó chỉ nguy hiểm khi chạy trên thread duy nhất chịu trách nhiệm timeout.
- Concurrency ở Judge xuất hiện từ một nhu cầu rất thực: **hai công việc phải tiến triển cùng lúc quanh cùng một child process**.
- Main và Reader độc lập về luồng chạy, nhưng không độc lập về hệ thống: chúng cùng phối hợp quản lý một process.

### Vấn đề mới lộ ra: drain tốt vẫn có thể giết Worker bằng RAM

Một user có thể in vô hạn:

```java
while (true) {
    System.out.println("A");
}
```

Nếu Reader append mọi byte vào `ByteArrayOutputStream`, pipe không đầy, nhưng RAM của host Judge sẽ lớn dần:

```text
pipe an toàn
  -> Judge RAM 10MB -> 100MB -> 1GB -> OOM
```

Ta đã sửa **pipe full**, nhưng sinh ra **host memory full**.

---

## 4. Phiên bản A+++ — output limit: bảo vệ RAM của Judge

### Vấn đề 4

Container memory limit không giới hạn số byte user có thể stream ra theo thời gian.

```text
Container --memory=128m
  bảo vệ RAM bên trong container

Host Judge ByteArrayOutputStream
  vẫn có thể tích 2GB stdout
```

Một program có thể dùng 20MB RAM nhưng in 2GB trong nhiều phút.

### A+++: Reader đếm byte và chỉ giữ output có giới hạn

```java
long bytesSeen = 0;
boolean outputExceeded = false;

while ((n = input.read(buffer)) != -1) {
    bytesSeen += n;

    if (bytesSeen > MAX_OUTPUT_BYTES) {
        outputExceeded = true;
        // policy: request termination
        process.destroyForcibly();
        break;
    }

    output.write(buffer, 0, n);
}
```

Đây chỉ là shape để hiểu flow. Phiên bản production sẽ cần synchronization và cleanup tốt hơn ở bước sau.

### Có hai policy hợp lệ sau khi vượt limit

#### Policy A — ngừng lưu, vẫn drain

```text
output vượt limit
  -> không append thêm vào RAM
  -> tiếp tục đọc/discard để pipe không đầy
  -> chờ process tự kết thúc hoặc timeout khác
```

#### Policy B — mark event, kill fail-fast

```text
output vượt limit
  -> record OUTPUT_LIMIT event
  -> yêu cầu terminate execution
  -> drain/await đủ để cleanup đúng cách
```

Với online judge, Policy B thường hợp lý: khi output vượt quota, testcase đã chắc chắn fail và không có ích gì để user tiếp tục dùng CPU. Nhưng policy phải được viết rõ và test rõ.

### Bản này giải quyết được gì?

- Bảo vệ RAM của Judge host.
- Phân biệt được ba resource problem khác nhau:

```text
Time limit   -> user chạy quá lâu
Memory limit -> user process dùng quá nhiều RAM
Output limit -> user gửi quá nhiều byte ra pipe
```

### Điều cần note

Không được làm thế này:

```text
vượt limit -> ngừng đọc hoàn toàn -> để child tiếp tục chạy
```

Vì pipe lại đầy và child block. Nếu không kill ngay, vẫn phải drain/discard.

### Vấn đề mới lộ ra: hai thread cần nói chuyện đúng cách

Reader phát hiện `outputExceeded=true`, còn Main cần dùng fact đó để ra verdict. Nếu dùng shared state hời hợt, ta gặp concurrency bugs.

---

## 5. Phiên bản A++++ — phối hợp Main và Reader đúng cách

### Vấn đề 5

Reader có thể phát hiện output limit gần cùng lúc Main thấy timeout/process exit. Shared state có ba câu hỏi khác nhau.

### 5.1 Ordering / timing

```text
Main đọc outputExceeded -> false
Reader sau đó mới set true
```

Đây không phải stale value. Tại thời điểm Main đọc, `false` là đúng.

**Cần giải quyết:** Main không được quyết định final verdict trước khi Reader hoàn thành phần evidence cần thiết.

**Cách thực tế:** await `Future<ReaderResult>`, `join`, hoặc `CountDownLatch` sau khi process được xử lý. Main chỉ final-evaluate sau khi nhận ReaderResult.

### 5.2 Visibility

```text
Reader đã set true trước
Main đọc sau
nhưng không có synchronization guarantee
```

Plain `boolean` không đủ để làm protocol giao tiếp giữa threads.

**Cách thực tế:**

```text
AtomicBoolean / volatile
synchronized / Lock
Future completion
CountDownLatch
```

`AtomicBoolean` hợp cho flag monotonic:

```text
false -> true
```

nhưng Future/result transfer thường tốt hơn khi cần chuyển cả output, byte count, exception, timestamp.

### 5.3 Atomicity

```java
counter = counter + 1;
```

Đây là read -> calculate -> write. Hai thread có thể cùng đọc 0 rồi cùng ghi 1: lost update.

**Cách thực tế:** `AtomicInteger.incrementAndGet()`, `synchronized`, lock, hoặc redesign ownership.

**Note:** atomicity problem không đồng nghĩa "lúc nào cũng phải dùng lock".

### A++++: ownership rõ ràng thay vì share bừa

```text
Reader owns
  - đọc bytes
  - mutate buffer cục bộ
  - đếm byte
  - phát hiện output-limit
  - trả ReaderResult immutable / signal event

Main owns
  - start / wait outer timeout / request kill
  - await ReaderResult
  - collect exit code và context
  - chọn final verdict duy nhất
  - persist/notify/cleanup
```

Một `ReaderResult` nên mang đủ evidence:

```text
capturedOutput
bytesSeen
outputExceeded
readerException
event timestamp/sequence (nếu policy cần)
```

### Bản này giải quyết được gì?

- Không để Reader và Main cùng ghi `status` rồi race nhau.
- Main đọc output sau khi Reader publish xong.
- Reader exception không bị thất lạc trong background thread.
- Concurrency trở thành protocol rõ ràng, không còn là vài biến boolean may rủi.

### Vấn đề mới lộ ra: exit code sau khi kill là mơ hồ

Reader phát hiện output limit rồi kill process. Main có thể thấy:

```text
done = true
exitCode = 137
outputExceeded = true
```

Nếu Main chỉ map non-zero -> `RUNTIME_ERROR`, verdict sai nguyên nhân.

Ta cần tách event/cause khỏi verdict.

---

## 6. Phiên bản A+++++ — event/cause trước, exit code sau

### Vấn đề 6

Exit code kể kết quả process termination, nhưng không luôn kể nguyên nhân nghiệp vụ mà Judge cần.

```text
137 có thể xuất hiện vì
  - Judge kill do output limit
  - Judge kill do timeout
  - OOM/container kill
  - external/infrastructure kill
```

### A+++++: record lifecycle events

Thay vì chỉ trả:

```text
RunResult(exitCode, output, duration)
```

hãy tư duy result giàu context hơn:

```text
ExecutionEvidence
  - processCompleted
  - directProcessExitCode
  - innerTimeoutObserved
  - outerTimeoutObserved
  - outputExceeded
  - oomKilled/containerState nếu runtime cung cấp được
  - judgeKillReason
  - readerFailure
  - capturedOutput
  - bytesSeen
  - duration
```

Reader và Main **record events**, còn Main là người áp policy để map sang verdict.

### Decision flow của một testcase

```text
1. Ensure process is terminated or outer failure is recorded.
2. Await reader(s), obtain ReaderResult and reader failure if any.
3. Combine all execution evidence.
4. Apply a documented verdict priority/cause policy.
5. Only then compare output if execution is otherwise valid.
```

Một thứ tự policy mẫu:

```text
specific outer/infrastructure failure?
  -> classify per product policy

outputExceeded?
  -> OUTPUT_LIMIT_EXCEEDED

trusted OOM evidence?
  -> MEMORY_LIMIT_EXCEEDED

inner timeout evidence / exit 124 from `timeout`?
  -> TIME_LIMIT_EXCEEDED

non-zero exit without more-specific cause?
  -> RUNTIME_ERROR

output differs expected?
  -> WRONG_ANSWER

otherwise
  -> testcase PASS
```

Thứ tự chính xác là **policy của product**, không phải định luật Java. Điều bắt buộc là nó phải deterministic, được viết rõ, và không phụ thuộc vào thread nào tình cờ chạy trước.

### Race: output limit và timeout xảy ra gần cùng lúc

Sai:

```text
Reader found it -> OUTPUT_LIMIT luôn thắng
Main found it -> TLE luôn thắng
```

Thread phát hiện là implementation detail, không phải business rule.

Các lựa chọn policy tốt:

- first terminal event wins: record timestamp/sequence, atomic claim terminal cause;
- fixed precedence: documented table;
- evidence-specific: ưu tiên direct observed cause hơn exit-code fallback mơ hồ.

### Bản này giải quyết được gì?

- `exitCode=137 + outputExceeded=true` đúng ra `OUTPUT_LIMIT_EXCEEDED`, không phải RE.
- `exitCode=0 + output khác expected` đúng ra WA.
- Một final verdict là kết quả của nhiều evidence, không phải một `if (exitCode != 0)`.

### Điều cần note thật kỹ

```text
exit code = symptom/result
event/state = cause/context
verdict = Judge classification
```

---

## 7. Phiên bản A++++++ — inner timeout + outer timeout (defense in depth)

### Vấn đề 7

Nếu chỉ có một timeout, nó thường không bảo vệ được mọi boundary.

- `timeout 2s java Main` bên trong container bảo vệ user code.
- Nhưng Docker CLI có thể treo, runtime có thể có lỗi, inner wrapper có thể không trả về, pipe/cleanup có thể bị lỗi.

### A++++++: hai lớp timeout có nhiệm vụ khác nhau

```text
INNER timeout
  `timeout 2s java Main < input.txt`
  -> time limit thật của bài cho user program
  -> expectation: timeout command reports exit 124

OUTER timeout
  `process.waitFor(10, SECONDS)` ở Worker host
  -> watchdog cho toàn Docker invocation
  -> protection against infrastructure/lifecycle failure
```

### Bản này giải quyết được gì?

- User infinite loop bị inner guard dừng theo limit bài.
- Worker không bị treo nếu inner path/Docker path có vấn đề.
- Đó là defense in depth: nhiều lớp độc lập, mỗi lớp bảo vệ một failure boundary.

### Điều dễ nhầm

```text
outer waitFor() returned true
  != user code chắc chắn pass time limit
```

Nó chỉ nói whole external process kết thúc trước outer deadline.

Ngược lại:

```text
outer timeout expired, but no inner-timeout evidence
```

không tự động chứng minh user code TLE. Có thể là Docker/worker infrastructure issue. Product cần định nghĩa cách hiển thị/lưu loại failure này; ít nhất phải giữ diagnostic context.

### Layer phòng thủ đầy đủ hơn

```text
inner timeout             -> execution time of user code
outer watchdog            -> Worker / Docker lifecycle
container memory / CPU    -> resource isolation of user process
network=none              -> network isolation
non-root user             -> privilege reduction
output cap                -> host Worker memory safety
bounded queue/concurrency -> host capacity safety
explicit cleanup          -> resource leak prevention
```

### Vấn đề mới lộ ra: `destroyForcibly()` không nhất thiết cleanup container đúng ý

Khi `ProcessBuilder` chạy executable `docker`, `Process` đại diện trực tiếp cho Docker CLI trên host. Kill CLI không có nghĩa ta đã directly kill `java Main` hay xác nhận container biến mất.

---

## 8. Phiên bản A+++++++ — Docker lifecycle và cleanup là một concern riêng

### Vấn đề 8

Business thought là "kill execution container", nhưng Java reality là:

```text
Java Worker
  -> Process handle
  -> docker CLI process on host
  -> Docker engine/runtime
  -> container
  -> bash
  -> java Main
```

### A+++++++: quản lý cleanup theo resource ownership

```text
On every path: pass / CE / RE / TLE / output limit / exception
  1. request termination if still alive
  2. confirm direct process termination
  3. cleanup/kill/remove known container if policy requires it
  4. wait/close reader tasks
  5. remove temporary work directory
  6. preserve diagnostic evidence needed for logs/result
```

`--rm` tốt nhưng không phải lý do để bỏ cleanup strategy. Hãy nghĩ recovery path nếu Docker CLI bị kill bất thường.

### Bản này giải quyết được gì?

- Không để zombie container, reader thread, hoặc temp directory dần làm hỏng host.
- Tách "kill direct child" khỏi "cleanup whole execution environment".

### Điều cần note

Security and resource limits là defense-in-depth; không có một flag Docker duy nhất làm Judge an toàn.

---

## 9. Từ một testcase lên submission: compile + aggregate

Đến đây ta có hàm khái niệm:

```text
runOneTestCase(input, expected, submission)
  -> ExecutionEvidence
  -> TestCaseResult / verdict
```

### Step 9.1 — compile là gate trước testcase

```text
write Main.java
  -> run `javac Main.java` trong sandbox
  -> compile failed?
       yes -> COMPILATION_ERROR, stop submission
       no  -> load/run testcases
```

Compile failure khác runtime failure: program chưa có executable user-code hợp lệ để chạy testcase.

### Step 9.2 — run từng testcase

```text
for each testcase in defined order
  -> write input.txt
  -> run sandbox execution with the safe pipeline A+++++++
  -> determine testcase verdict from evidence
```

### Step 9.3 — aggregate theo fail-fast policy

```text
Test 1 -> PASS
Test 2 -> PASS
Test 3 -> WRONG_ANSWER
  -> Submission = WRONG_ANSWER
  -> stop; Test 4 need not run
```

```text
Submission = ACCEPTED
  only if every testcase passes.
```

### Bản này giải quyết được gì?

- Không nhầm one-testcase AC với whole-submission AC.
- Giữ evidence/result theo testcase để debugging và sau này có thể hỗ trợ richer feedback.
- Chỉ persist/send một final submission status sau aggregation.

### Điều cần note

```text
TestCaseResult
  = result of one input

SubmissionResult
  = aggregate business result across all cases
```

---

## 10. Hình dạng của bản cuối (không phải code copy-paste)

Sau tất cả nâng cấp, flow nên đọc được như sau:

```text
Receive submission job
  -> mark RUNNING
  -> create isolated work directory
  -> compile in sandbox through safe execution wrapper
  -> if compile evidence says failure: finish COMPILATION_ERROR
  -> for each testcase
       -> write input
       -> start Docker execution
       -> immediately start bounded output drainer(s)
       -> concurrently enforce outer watchdog
       -> inner sandbox timeout enforces user time limit
       -> record output-limit / timeout / OOM / kill / reader-error events
       -> terminate and cleanup when policy requires
       -> await reader result
       -> evaluate evidence once in Main/Judge
       -> if failure: finish submission with that verdict
  -> all testcase passes: finish ACCEPTED
  -> persist, notify, cleanup
```

### Invariants trước khi gọi bản này là "Judge Worker an toàn hơn"

- Stream draining bắt đầu ngay sau `pb.start()`.
- Captured output bị bound theo số byte.
- Inner user-code timeout và outer Worker timeout không bị trộn ý nghĩa.
- Reader event/result được publish với synchronization rõ ràng.
- Main là owner duy nhất của final verdict.
- Exit code không được dùng như source of truth duy nhất cho cause.
- Main chỉ compare output sau khi execution valid và reader complete.
- Mọi terminal path cleanup được process/container/temp-dir/reader resources.
- Submission chỉ AC nếu tất cả testcase pass.

---

## 11. Checklist học / test theo đúng thứ tự phát triển

Không test bản cuối bằng một happy path duy nhất. Test từng bước tương ứng với bug đã buộc ta nâng cấp:

1. Command nhỏ in stdout, stderr, exit 7.
   - Verify ProcessBuilder arguments, stream direction, exit code.
2. Infinite loop không output.
   - Verify `waitFor(timeout)` + kill và ý nghĩa `done=false`.
3. Infinite loop/high-output.
   - Verify timeout monitor vẫn chạy trong lúc reader drain.
4. Child chạy nhanh nhưng in output lớn.
   - Verify không có false TLE do pipe full.
5. Infinite output vượt quota.
   - Verify host memory không tăng vô hạn; policy kill/discard đúng; pipe không bị bỏ mặc.
6. Reader thấy output limit và Main thấy process ended/timeout gần đồng thời.
   - Verify only one verdict and policy deterministic.
7. Inner timeout 2s.
   - Verify event/exit 124 maps to TLE.
8. Container OOM-like/resource failure và external kill.
   - Verify exit 137 không bị hard-code mù quáng; verdict uses evidence.
9. Exit 0 nhưng output sai.
   - Verify WA, không phải AC.
10. Compile error.
   - Verify CE stops testcase loop.
11. Multi-test submission with test 3 failure.
   - Verify aggregation/fail-fast and final status.
12. Exception during Docker/read/cleanup.
   - Verify cleanup and explicit infrastructure/worker error path.

---

## 12. Năm câu để tự recall sau này

1. Bản A sai ở đâu khi child chạy vô hạn?
2. A+ sửa timeout nhưng tạo TLE giả thế nào khi child in rất nhiều output?
3. A++ sửa pipe blocking, nhưng tại sao vẫn có thể OOM host Judge?
4. A+++ có `outputExceeded`; tại sao A++++ vẫn phải có Future/latch/atomic ownership thay vì boolean thường?
5. Sau khi có mọi event, tại sao `exitCode` vẫn chỉ là một evidence chứ không phải final verdict?

Nếu trả lời được chuỗi năm câu này bằng flow nguyên nhân -> cải tiến -> vấn đề mới, bạn đang hiểu Judge Worker như một hệ thống, không chỉ nhớ API Java.

---

## Source trail

- Session “Branch · Giải thích ProcessBuilder”: đường phát triển ProcessBuilder -> timeout -> pipe blocking -> concurrent draining -> output limit -> concurrency -> verdict policy.
- `backend-plan.md`: Phase 5 Docker image/constraints, inner `timeout 2s`, outer wait, compilation/testcase/submission flow.
- `LEARNING_LOG.md`: project learning context.
