# 🔷 Lời đề tựa

## Tại sao chọn Roc?

Hãy tưởng tượng một ngôn ngữ lập trình mà:

- ❌ **Không có null** — không bao giờ "null pointer exception"
- ❌ **Không có exceptions** — mọi lỗi đều hiện rõ trong code
- ❌ **Không có side effects lén lút** — function thuần luôn cho cùng kết quả
- ✅ **Compiler là bạn đồng hành** — nó bảo bạn trước khi code chạy

Đó là **Roc** — ngôn ngữ functional programming thuần khiết, biên dịch sang native binary, với triết lý "simple over complex".

Cuốn sách này kết hợp **bốn cuốn sách kinh điển**:

| Nguồn | Đóng góp |
|-------|----------|
| **Domain Modeling Made Functional** (Scott Wlaschin) | DDD + Type-driven design |
| **FP Made Easier** (Charles Scalfani) | Nền tảng FP từ zero |
| **Learn Go with Tests** (Chris James) | TDD methodology |
| **F# for Fun and Profit** (Scott Wlaschin) | ROP, Patterns, Practical FP |

...và chuyển thể sang **Roc** — ngôn ngữ inherently pure, với hệ thống Platform tách biệt IO khỏi logic, giúp bạn viết code *đúng từ đầu*.

## Roc khác gì với ngôn ngữ khác?

| Đặc điểm | Roc | Rust/Go/Python |
|-----------|-----|----------------|
| Pure by default | ✅ Mọi function đều pure | ❌ Phải tự enforce |
| Side effects | Qua Platform, rõ ràng (`!`) | Bất kỳ đâu, ngầm |
| Null/nil | ❌ Không tồn tại | ✅ Có (và hay crash) |
| Exceptions | ❌ Dùng `Result` | ✅ Có try/catch |
| Testing | `expect` inline ngay trong code | File riêng |
| GC pauses | ❌ Reference counting | ✅/❌ Tùy ngôn ngữ |

## Cuốn sách này dành cho ai?

- 🧒 **Người mới hoàn toàn** — Roc có cú pháp đơn giản, ít "ceremony"
- 🎓 **Developer web/backend** — muốn chuyển sang tư duy Functional
- 🏗️ **FP practitioner** — muốn DDD patterns trong pure functional language
- 🔬 **Người tò mò** — muốn hiểu tại sao "no side effects" lại là lợi thế

## Triết lý viết sách

> *"Code không chỉ chạy ra kết quả đúng — code phải KHÔNG THỂ chạy ra kết quả sai."*

Mỗi chapter tuân theo nguyên tắc:

1. **Ẩn dụ quen thuộc** — "Platform giống như nhà bếp — bạn nấu, bếp cung cấp gas, nước, điện"
2. **Roc trước, lý thuyết sau** — thấy code chạy trước, hiểu tại sao sau
3. **Inline testing** — `expect` ngay cạnh code, không cần file test riêng
4. **Platform = safety** — bạn KHÔNG THỂ làm IO mà platform không cho phép
5. **Checkpoint kiểm tra** — dừng lại, tự hỏi "mình hiểu chưa?"
6. **Troubleshooting** — lỗi thường gặp ở cuối mỗi chapter
7. **Dự án xuyên suốt** — domain "quán café" xuất hiện từ Ch7 đến Capstone (Ch26), giúp bạn thấy sự tiến triển: từ records đơn giản → state machine → event sourcing → CLI app hoàn chỉnh

---

# 📖 Hướng dẫn đọc sách

## Cấu trúc sách

```
    +--------------------------------------------+
    |       Chapter 0: Roc in 10 Minutes          |
    |   (Syntax toi thieu de bat dau - 10 phut)   |
    +----------------------+---------------------+
                           |
                           v
    +--------------------------------------------+
    |         PART 0: CS Foundations              |
    |   Ch 1-3  |  Math, Algorithms, Data Struct  |
    |   Level: Pre-requisite                      |
    +----------------------+---------------------+
                           |
                           v
    +--------------------------------------------+
    |        PART 1: Roc Fundamentals             |
    |   Ch 4-10  |  Values, Functions, Records,   |
    |              Pattern Matching, Opaque Types |
    |   Level: Beginner                           |
    +----------------------+---------------------+
                           |
                           v
    +--------------------------------------------+
    |       PART 2: Thinking Functionally         |
    |   Ch 11-14  |  Purity, Tasks, Errors,       |
    |               Abilities                     |
    |   Level: Intermediate                       |
    +----------------------+---------------------+
                           |
                           v
    +--------------------------------------------+
    |        PART 3: Design Patterns              |
    |   Ch 15-16  |  FP Patterns,                 |
    |               Event Sourcing                |
    |   Level: Intermediate                       |
    +----------------------+---------------------+
                           |
                           v
    +--------------------------------------------+
    |         PART 4: DDD with Roc                |
    |   Ch 17-21  |  Domain Modeling, Workflows,  |
    |               Platform Separation           |
    |   Level: Advanced                           |
    +----------------------+---------------------+
                           |
                           v
    +--------------------------------------------+
    |      PART 5: Testing & Applications         |
    |   Ch 22-26  |  TDD, PBT, CLI, Web,          |
    |               Capstone                      |
    |   Level: Advanced                           |
    +----------------------+---------------------+
                           |
                           v
    +--------------------------------------------+
    |      PART 6: Production Engineering         |
    |   Ch 27-33  |  Database, Security,          |
    |               System Design, Capstone       |
    |   Level: Principal                          |
    +--------------------------------------------+
```

## Bạn nên bắt đầu từ đâu?

### 🟢 Người mới bắt đầu (Beginner)

> *"Tôi chưa biết lập trình, hoặc chưa biết Roc."*

```
    Ch0 ------> Ch1-3 ------> Ch4-10 ------> Dừng lại, làm bài tập
    10 min      Nền tảng       Roc cơ bản     Tổng: ~2 tuần
```

**Cách đọc:**
- Đọc **TẤT CẢ** từ Chapter 0, **theo thứ tự**
- Mỗi ngày đọc **1 chapter**, chạy code bằng `roc run`
- Gọi `roc test` sau mỗi bài tập để kiểm tra
- Gặp Checkpoint → trả lời trước khi mở đáp án
- **Đặc biệt chú ý Ch7** (Records & Tag Unions) — đây là trái tim của Roc

### 🔵 Developer có kinh nghiệm (Intermediate)

> *"Tôi biết Python/JS/Go, muốn học FP với Roc."*

```
    Ch0 ----> Skim Ch1-3 ----> Ch4-10 ----> Ch11-16
    10 min    Nếu biết          Roc syntax   FP Thinking
              basics: skim                   + Patterns
```

**Cách đọc:**
- Ch0: syntax overview, 10 phút
- Part 0: skim nhanh nếu đã biết Big-O, recursion
- Part 1: đọc kỹ **Ch7 (Tag Unions)** và **Ch10 (Opaque Types)** — khác mọi ngôn ngữ bạn biết
- **Part 2 là điểm chuyển biến** — purity, Tasks, `!` suffix sẽ thay đổi cách bạn nghĩ về code
- Bài tập: làm hết — Roc syntax mới, cần luyện tay

### 🟣 FP Developer (Advanced)

> *"Tôi biết Elm/Haskell/F#, muốn DDD patterns trong Roc."*

```
    Ch0 ----> Ch7+10 -----> Ch11-14 ----> Ch17-26
    Syntax    Tag Unions     Purity +      DDD + Apps
    10 min    + Opaque       Abilities     FOCUS HERE
```

**Cách đọc:**
- Bạn đã biết FP → skim Part 0-1, **chỉ đọc kỹ Roc-specific features**:
  - **Ch7**: tag unions = structural (khác Elm/Haskell nominal)
  - **Ch10**: opaque types + abilities (khác typeclasses)
  - **Ch12**: Tasks + `!` = Roc's Effect system
- **Part 4-5: ĐÂY LÀ PHẦN CHÍNH** — DDD trong pure FP context
- **Ch20 Platform Separation** = concept hoàn toàn mới (không có trong Elm/Haskell)
- Xem **Appendix B** (Elm/Gleam Translation) để mapping nhanh

### 🔴 Architect / Principal

> *"Tôi muốn system design, production patterns."*

```
    Ch0 ----> Ch7+12 -----> Ch17-21 ----> Ch27-33
    Syntax    Core Roc      DDD core      PRODUCTION
    10 min    features      patterns      The goal
```

**Cách đọc:**
- Ch0 + Ch7 (Tags) + Ch12 (Tasks) = hiểu Roc đủ sâu, 1 ngày
- Part 4 (DDD): focus vào **Ch20 Platform Separation** — unique architecture
- **Part 6: ĐÂY LÀ MỤC TIÊU** — DB, Security, Distributed Systems
- Capstone (Ch33): xem Roc trong production context

---

## Cách Roc thay đổi tư duy

```
    TRƯỚC KHI ĐỌC SÁCH:             SAU KHI ĐỌC SÁCH:

    "Code CÓ THỂ crash"             "Code KHÔNG THỂ crash"
          |                                |
          v                                v
    try {                           validate = \input ->
      data = fetch()                  when parse input is
      process(data)                     Ok order -> Ok order
    } catch (e) {                       Err err -> Err err
      // maybe handle?
      // maybe not?                 process = \order ->
    }                                 order
                                        |> validate
    "Ai biết function này               |> Result.try price
     có side effect không?"             |> Result.try save

    "Null check ở đâu nhỉ?"         "Compiler đã check hết.
                                     Nếu compile = chạy đúng."
```

---

## Quy ước trong sách

| Ký hiệu | Ý nghĩa |
|----------|---------|
| 💡 | Mẹo hoặc insight quan trọng |
| ⭐ | Chapter đặc biệt quan trọng |
| ✅ Checkpoint | Dừng lại tự kiểm tra |
| 🏋️ Bài tập | Thực hành (có lời giải) |
| 🔧 Troubleshooting | Lỗi thường gặp + cách sửa |
| 📋 Tóm tắt | Tổng hợp kiến thức cuối chapter |
| → Tiếp theo | Link sang chapter tiếp |
| `!` trong code | Effectful call (side effect) |

## Code trong sách

```roc
# Code Roc trong sách dùng quy ước:
# - Tên biến/function: tiếng Anh (chuẩn Roc)
# - Comment giải thích: tiếng Việt
# - expect = test inline, chạy bằng roc test

# Tính diện tích hình tròn
circleArea = \radius ->
    Num.pi * radius * radius

# Tests ngay cạnh code!
expect circleArea 5.0 == 78.53981633974483
expect circleArea 0.0 == 0.0

# Effectful code dùng !
main =
    name = Stdin.line!           # đọc input từ user
    Stdout.line! "Hello, $(name)!" # in ra màn hình
```

## Thời gian ước tính

| Trình độ | Phần đọc | Thời gian |
|----------|----------|-----------|
| Beginner | Part 0-1 (Ch0-10) | 1.5-2 tuần |
| Intermediate | Part 2-3 (Ch11-16) | 1 tuần |
| Advanced | Part 4-5 (Ch17-26) | 2-3 tuần |
| Principal | Part 6 (Ch27-33) | 1.5-2 tuần |
| **Toàn bộ sách** | **Ch0-33** | **~6-8 tuần** |

> **Lời khuyên cuối**: Roc nhỏ nhưng sâu. Cú pháp ít, nhưng mỗi concept là một tảng đá vững chắc. Đừng vội — hãy để compiler dẫn đường cho bạn.

---

*Chúc bạn khám phá niềm vui của "code không thể trả ra kết quả sai"!* 🔷
