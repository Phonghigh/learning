# Concept note: `dataclasses.field(default_factory=list)` — tránh mutable default dùng chung

Không phải một session debug đầy đủ — đây là ghi chú khái niệm phát sinh khi đọc code Python trong
EWM-App (dataclass import mạng SWMM, field `junctions: list[JunctionRecord] = field(default_factory=list)`).

## Vấn đề cốt lõi

Trong Python, giá trị mặc định của tham số hàm (hoặc field trong dataclass) chỉ được **tạo ra một
lần duy nhất**, tại thời điểm hàm/class được **định nghĩa** — không phải mỗi lần gọi/khởi tạo.

Với hàm thường:

```python
def add_item(item, bucket=[]):   # bucket=[] tạo MỘT LẦN khi def chạy
    bucket.append(item)
    return bucket

add_item(1)   # [1]
add_item(2)   # [1, 2]  <-- bug: dùng chung list với lần gọi trước!
```

Cả hai lần gọi cùng tham chiếu đến **cùng một object list** trong bộ nhớ, vì `[]` không được tạo lại
mỗi lần `add_item` chạy — Python không hề âm thầm cảnh báo gì cả.

## Vì sao dataclass "bảo vệ" bạn tốt hơn hàm thường

```python
from dataclasses import dataclass

@dataclass
class Bad:
    items: list = []   # ValueError khi định nghĩa class, không đợi tới lúc chạy
```

`dataclass` chủ động phát hiện default là kiểu mutable (`list`, `dict`, `set`) và raise
`ValueError: mutable default <class 'list'> for field items is not allowed: use default_factory`
ngay khi class được định nghĩa — đây là điểm khác biệt quan trọng so với hàm thường (`def f(x=[])`),
vốn không báo lỗi gì và âm thầm gây bug về sau.

## Giải pháp: `default_factory`

```python
from dataclasses import dataclass, field

@dataclass
class Good:
    items: list = field(default_factory=list)
```

`default_factory` nhận một **callable không tham số** — Python sẽ **gọi hàm đó mỗi lần khởi tạo
instance mới** để tạo ra một object mới, độc lập, thay vì tái sử dụng một object có sẵn từ lúc định
nghĩa class.

`default_factory` không giới hạn ở `list` — có thể dùng:

```python
field(default_factory=dict)
field(default_factory=set)
field(default_factory=lambda: [1, 2, 3])   # callable tùy ý, không tham số
```

## Ví dụ thực tế từ EWM-App

```python
@dataclass
class SwmmNetwork:
    junctions: list[JunctionRecord] = field(default_factory=list)
    conduits: list[ConduitRecord] = field(default_factory=list)
```

Nếu viết `junctions: list[JunctionRecord] = []` thay vào đó, mỗi `SwmmNetwork()` mới sẽ raise
`ValueError` ngay lập tức khi module được import — dataclass chặn bug này từ trước khi code chạy.
Nếu (giả sử) không dùng dataclass mà tự viết `__init__(self, junctions=[])`, thì mọi instance
`SwmmNetwork` sẽ **dùng chung một list junctions** — network A thêm junction thì network B cũng thấy
junction đó xuất hiện, dù trông như hai object độc lập.

## Cách tìm lại

- Grep: `default_factory`, `field(default_factory`, `mutable default`
- Tên lỗi để tra cứu: `ValueError: mutable default <class 'list'> for field ... is not allowed`
- Khái niệm liên quan (đọc thêm nếu gặp lại): "mutable default argument" — đây là bug kinh điển của
  Python, không riêng gì dataclass, áp dụng cho MỌI default argument kiểu mutable trong hàm thường.

## Gotchas

- Bug này **không giới hạn ở `dataclass`** — bất kỳ `def f(x=[])`, `def f(x={})` nào cũng dính, và
  Python không báo lỗi gì trong trường hợp hàm thường. Chỉ `dataclass` mới chủ động raise.
- `default_factory` phải là callable **không nhận tham số nào** — nếu cần truyền tham số vào factory,
  phải bọc trong `lambda: some_func(arg)`.
- Đối tượng "an toàn" làm default trực tiếp (không cần `field()`) là các kiểu **immutable**: `int`,
  `str`, `float`, `bool`, `None`, `tuple` (nếu tuple chỉ chứa phần tử immutable) — vì chúng không thể
  bị mutate sau khi tạo, nên chia sẻ giữa các instance không gây bug.
- Tương tự với `dict` và `set` — cùng nguyên tắc, cùng lỗi `ValueError`, cùng cách sửa bằng
  `field(default_factory=dict)` / `field(default_factory=set)`.
