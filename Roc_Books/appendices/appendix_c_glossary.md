# Appendix C — Bảng thuật ngữ Anh-Việt (Glossary)

> Tổng hợp thuật ngữ kỹ thuật sử dụng xuyên suốt cuốn sách.

---

## Thuật ngữ Roc

| English | Tiếng Việt | Giải thích ngắn |
|---------|-----------|-----------------|
| Ability | Khả năng | Interface tối thiểu: `Eq`, `Hash`, `Inspect`, `Encode`, `Decode` |
| App | Ứng dụng | Phần code pure của bạn — không chứa IO trực tiếp |
| Backpassing (`<-`) | Truyền ngược | Sugar syntax cho callback: `x <- Task.await task` |
| Bang suffix (`!`) | Dấu chấm than | Đánh dấu effectful call: `Stdout.line!` |
| Closed tag union | Union đóng | Chính xác các tags này, không hơn: `[Red, Green, Blue]` |
| Crash | Sập | Roc không có runtime crashes (trừ platform bugs) |
| Currying | Curry hóa | Tự động: `add 3` trả function nhận 1 tham số nữa |
| Destructuring | Phân rã | Mở record/tag lấy fields: `\{ name, age } -> ...` |
| Expect | Kỳ vọng | Inline test: `expect add 2 3 == 5` |
| Module | Mô-đun | File `.roc` expose functions/types: `module [fn1, Type1]` |
| Opaque type | Kiểu chắn | Type bọc ngoài, che data: `Email := Str` |
| Open record | Record mở | "Có ít nhất fields này": `{ name : Str }a` |
| Open tag union | Union mở | "Có ít nhất tags này": `[Red, Green]a` |
| Pattern matching | Khớp mẫu | `when x is Red -> ... Green -> ...` |
| Pipeline (`|>`) | Ống dẫn | Chaining: `x |> f |> g` = `g (f x)` |
| Platform | Nền tảng | Runtime xử lý IO: `basic-cli`, `basic-webserver` |
| Pure function | Hàm thuần | Không side effects — cùng input luôn ra cùng output |
| Record | Bản ghi | Nhóm data: `{ name: "An", age: 25 }` |
| Record update | Cập nhật bản ghi | Tạo bản mới: `{ old & field: newValue }` |
| Structural typing | Kiểu cấu trúc | Type theo hình dạng, không theo tên khai báo |
| Tag | Thẻ | Giá trị có tên: `Ok`, `Err`, `Red`, `Confirmed` |
| Tag union | Kiểu hợp | Một trong nhiều: `[Red, Green, Blue]` |
| Task | Tác vụ | Mô tả side effect: `Task ok err` |
| Type inference | Suy luận kiểu | Compiler tự đoán type — ít cần annotation |

---

## Thuật ngữ Functional Programming

| English | Tiếng Việt | Giải thích ngắn |
|---------|-----------|-----------------|
| Algebraic Data Types (ADT) | Kiểu dữ liệu đại số | Product (records, AND) + Sum (tag unions, OR) |
| Closure | Bao đóng | Function "nhớ" biến ngoài scope |
| Composition | Hợp thành | Kết hợp functions: `f >> g` hoặc `x |> f |> g` |
| Fold / Reduce | Gấp / Rút gọn | `List.walk` — duyệt list, tích lũy kết quả |
| Higher-order function | Hàm bậc cao | Function nhận/trả function: `List.map items \x -> ...` |
| Immutability | Bất biến | Data không thay đổi sau khi tạo |
| Lambda | Hàm vô danh | `\x -> x + 1` |
| Map | Ánh xạ | Biến đổi mỗi phần tử: `List.map` |
| Partial application | Áp dụng bán phần | `add 3` → function chờ 1 arg nữa |
| Persistent data structure | Cấu trúc bền vững | Cập nhật tạo bản mới, bản cũ giữ nguyên |
| Pure core / Effectful shell | Lõi thuần / Vỏ hiệu ứng | Pattern: logic pure ở giữa, IO ở ngoài |
| Railway-Oriented Programming (ROP) | Lập trình theo đường ray | Chaining `Result.try` — happy path + error path |
| Recursion | Đệ quy | Function gọi chính nó |
| Reference counting | Đếm tham chiếu | Quản lý bộ nhớ tự động — Roc dùng thay GC |
| Side effect | Hiệu ứng phụ | Thay đổi thế giới bên ngoài (IO, file, HTTP) |
| Tail-call optimization (TCO) | Tối ưu đệ quy đuôi | Đệ quy ở vị trí cuối → không tốn stack |
| Total function | Hàm toàn phần | Trả kết quả cho MỌI input — không crash |

---

## Thuật ngữ Domain-Driven Design

| English | Tiếng Việt | Trong Roc |
|---------|-----------|-----------|
| Aggregate | Tập hợp gốc | Record chứa entities + value objects |
| Anti-Corruption Layer | Lớp chống hư hỏng | Parse external data → domain types (`Decode`) |
| Bounded Context | Ngữ cảnh giới hạn | Module Roc riêng biệt |
| Command | Lệnh | Tag union: `[CreateOrder {...}, CancelOrder {...}]` |
| Domain Event | Sự kiện miền | Tag union: `[OrderPlaced {...}, OrderShipped {...}]` |
| Entity | Thực thể | Record có `id` field — identity qua id |
| Event Sourcing | Nguồn sự kiện | Lưu events → `List.walk` rebuild state |
| Onion Architecture | Kiến trúc củ hành | Pure core (domain) + Effectful shell (IO) |
| Projection | Chiếu | Function từ events → view model |
| Snapshot | Bản chụp | Cache state tại 1 thời điểm — tối ưu rebuild |
| State Machine | Máy trạng thái | Tag union cho states + transition functions |
| Ubiquitous Language | Ngôn ngữ chung | Code names = business names |
| Value Object | Đối tượng giá trị | Opaque type: `Email := Str`, `Money := U64` |
| Workflow | Quy trình | Pipeline: `validate |> process |> save` |

---

## Thuật ngữ Production Engineering

| English | Tiếng Việt | Giải thích ngắn |
|---------|-----------|-----------------|
| ACID | ACID | Atomicity, Consistency, Isolation, Durability |
| B-Tree | Cây B | Cấu trúc index database — tìm O(log n) |
| CAP Theorem | Định lý CAP | Consistency, Availability, Partition tolerance — chọn 2 |
| Circuit Breaker | Cầu dao | Ngắt request khi service lỗi liên tục |
| CORS | CORS | Cross-Origin Resource Sharing — security headers |
| CSRF | CSRF | Cross-Site Request Forgery — attack giả mạo request |
| CTE | CTE | Common Table Expression — "let binding" trong SQL |
| JWT | JWT | JSON Web Token — stateless authentication |
| Normalization | Chuẩn hóa | Tổ chức DB tránh trùng lặp data |
| OAuth 2.0 | OAuth 2.0 | Authorization protocol — login qua Google/GitHub |
| OWASP | OWASP | Open Web Application Security Project — top 10 vulnerabilities |
| RBAC | RBAC | Role-Based Access Control — quyền theo vai trò |
| Saga | Saga | Distributed transaction pattern — compensating actions |
| XSS | XSS | Cross-Site Scripting — inject code vào web page |
