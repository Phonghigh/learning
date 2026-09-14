# Series overview

## Working title

Building LeetCodeFake — Từ một bài submit đến Online Judge

## Main story

Tôi tự xây một LeetCode Clone không chỉ để clone sản phẩm. Tôi muốn hiểu thật rõ chuyện gì xảy ra sau nút Submit: code được compile ở đâu, process được tạo thế nào, container bảo vệ resource nào, thread phối hợp ra sao, và Judge quyết định Accepted dựa trên evidence nào.

Series đi theo quá trình build thật:

~~~text
bản đơn giản → failure mode → hiểu nguyên nhân → nâng cấp nhỏ nhất → failure mode mới
~~~

## Content arcs

### Arc 1 — Bắt đầu từ bài toán nhỏ

Động lực, project boundary, một testcase, external command và distinction giữa exit code với verdict.

### Arc 2 — Judge Worker evolution

Timeout, pipe backpressure, reader thread, output limit, shared state, lifecycle coordination và termination reason.

### Arc 3 — Mở rộng thành Online Judge

Compile, nhiều testcase, Docker lifecycle, inner/outer timeout, cleanup và submission-level aggregation.

## Writing principles

- Kể theo project journey, không biến thành textbook.
- Một bài chỉ có một thay đổi kỹ thuật chính.
- Luôn cho thấy code before/after khi có evolution.
- Không gọi exitCode == 0 là Accepted.
- Không gọi conceptual design là production-ready.
- Cuối bài luôn chỉ ra vấn đề buộc bài sau phải xuất hiện.

