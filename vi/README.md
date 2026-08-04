# Cyclone Protocol

[English](../README.md) · **Tiếng Việt**

Cyclone là một cách quy định **dữ liệu được viết thành byte như thế nào**.

Chỉ vậy thôi.

---

## Giải thích trong 30 giây

Bạn có một `Player`:

```
Hp   = 100
Name = "Alice"
```

Cyclone quy định nó thành đúng chuỗi byte này:

```
64 00 00 00   05 00 00 00   41 6C 69 63 65
   Hp = 100    Length = 5      A  l  i  c  e
```

13 byte. Không có tên field, không có dấu ngoặc, không có thẻ đánh dấu — chỉ có dữ liệu.

Điểm mấu chốt: **Go, C#, Rust hay C đều phải sinh ra đúng 13 byte này.** Không phải "tương đương", mà là giống hệt từng byte.

---

## Vì sao lại cần điều đó

Với JSON hay Protobuf, cùng một dữ liệu có thể ra byte khác nhau tuỳ ngôn ngữ, tuỳ thư viện, tuỳ version. Thường thì không sao. Nhưng nó hỏng hẳn khi:

- Bạn **ký hoặc hash** payload — chữ ký ký lên byte, byte đổi thì chữ ký sai dù dữ liệu không đổi.
- Bạn chạy **replay hoặc lockstep** — hai máy phải ra cùng một kết quả từ cùng một chuỗi input.
- Bạn muốn **kiểm chứng hai SDK tương thích** — cách duy nhất là so byte.

Cyclone giải bài toán này bằng cách bỏ hết lựa chọn: chỗ nào encoder được quyền chọn, chỗ đó hai encoder sẽ chọn khác nhau.

---

## Cyclone không phải cái gì

```
✗ Networking framework    ✗ RPC
✗ Game engine             ✗ Transport (TCP/UDP/QUIC)
✗ Encryption              ✗ Compression
```

Những thứ đó xây **phía trên** Cyclone. Cyclone chỉ đứng đúng ở chỗ dữ liệu đổi hình dạng:

```
Model  →  Bytes
Bytes  →  Model
```

---

## Cách nó hoạt động

Bốn quy tắc đủ để hiểu toàn bộ:

**1. Số có kích thước cố định, little-endian.** `UInt32` luôn 4 byte, kể cả giá trị 0. Không varint.

**2. Chuỗi và mảng có tiền tố độ dài.** `[4 byte length][dữ liệu]`. Với chuỗi, length là **số byte UTF-8**, không phải số ký tự.

**3. Struct là các field nối liền nhau.** Không header, không padding, không delimiter. Decoder biết đang đọc field nào **nhờ vị trí con trỏ**, không nhờ tag.

**4. Không có metadata.** Byte stream không tự mô tả. Bên nhận **bắt buộc** phải biết trước đúng thứ tự và kiểu của từng field.

Hệ quả của quy tắc 3 và 4, cần nắm trước khi dùng:

```
Đổi TÊN field      →  không đổi byte nào        (an toàn)
Đổi THỨ TỰ field   →  đổi toàn bộ byte stream   (breaking change)
Sai định nghĩa     →  thường KHÔNG báo lỗi, mà ra dữ liệu sai im lặng
```

---

## Cyclone không có Schema Definition riêng

Không có IDL. Không có file `.cyclone`. Không có tài liệu schema phải biên dịch trước.

**Model của bạn, đã đánh dấu annotation, chính là định nghĩa duy nhất.**

```csharp
[Network]
public partial class Player
{
    [Network(UInt32)]
    public uint Hp;

    [Network(PlayerInfo)]
    public PlayerInfo Info;
}
```

Cyclone không cần hiểu `public`, `partial`, `class`, `uint`, hay `PlayerInfo` là class hay struct. Nó chỉ cần trích ra đúng ba thứ:

```
Type Name    →  Player
Field Name   →  Hp
Cyclone Type →  UInt32
```

Annotation là **nguồn sự thật duy nhất**. Nếu bạn viết `[Network(UInt32)]` trên một field kiểu khác, Cyclone vẫn sinh theo `UInt32` — và báo lỗi nếu không khớp. Đó là lỗi khai báo, không phải chuyện Cyclone phải đoán.

Hệ quả: viết frontend cho một ngôn ngữ mới rất nhẹ. Mỗi ngôn ngữ chỉ cần đọc được annotation + tên type + tên field, không cần semantic analysis, không cần reflection, không cần hiểu type system của ngôn ngữ đó.

---

## Khi nào KHÔNG nên dùng

Nói trước cho rõ:

- Cần client cũ nói chuyện được với server mới → **Cyclone không hợp**. v1 không có schema evolution.
- Schema đổi liên tục → mỗi lần đổi phải deploy đồng bộ cả hai phía.
- Cần đọc dữ liệu bằng mắt → Cyclone là nhị phân thuần.
- Có nhiều field "có thể không có" → v1 không có Optional/Nullable.

Nếu bạn thuộc các nhóm trên, Protobuf là lựa chọn đúng hơn.

---

## Điều Cyclone đảm bảo

```
Cùng schema, cùng giá trị

↓

Mọi implementation MUST sinh ra byte giống hệt nhau.
```

Nếu không giống, đó là **bug của implementation**, không phải của protocol. Và bug đó bắt được bằng một phép so sánh mảng byte — xem tài liệu Conformance.

---

## Tài liệu

Đọc theo thứ tự này:

| Tài liệu | Trả lời câu hỏi |
|----------|-----------------|
| [RFC-0001 — Cyclone là gì](RFC-0001.md) | Giải bài toán gì, vì sao chọn, khi nào **không** nên dùng |
| [RFC-0002 — Wire Format](RFC-0002.md) | Byte phải có hình dạng như thế nào |
| [RFC-0003 — Conformance](RFC-0003.md) | Làm sao biết implementation của tôi đúng (test vector) |

Bản dịch tiếng Anh: [`en/RFC-0001.md`](../en/RFC-0001.md) · [`en/RFC-0002.md`](../en/RFC-0002.md) · [`en/RFC-0003.md`](../en/RFC-0003.md)

Trang giới thiệu: [cyclone-protocol.github.io/cyclone](https://cyclone-protocol.github.io/cyclone/) — nguồn tại [`index.html`](../index.html) (tiếng Anh) và [`vi/index.html`](index.html) (tiếng Việt)

---

## Đóng góp

Cyclone là **specification**, không phải thư viện. Cần implementation cho Rust, Go, C#/Unity, C/embedded, Zig/C++.

Tiêu chí duy nhất để được gọi là **Cyclone Compatible**:

```
Chạy test vector trong RFC-0003

↓

Pass 100%
```

Không có 98%. Một vector fail nghĩa là tồn tại một dữ liệu mà implementation của bạn và implementation khác bất đồng.

Xem [bảng trạng thái ecosystem](https://cyclone-protocol.github.io/cyclone/#implementations) để biết ngôn ngữ nào đang trống.

### Bản dịch

Tiếng Việt là bản gốc, tiếng Anh là bản dịch. Quy trình mô tả trong [`TRANSLATION.md`](../TRANSLATION.md).

---

## Cấu trúc repo

```
.
├── README.md            tài liệu này (tiếng Anh)
├── TRANSLATION.md       hai bản ngôn ngữ giữ đồng bộ như thế nào
├── index.html           landing page, tiếng Anh (GitHub Pages)
├── LICENSE              CC BY 4.0
│
├── vi/                  tiếng Việt — bản gốc
│   ├── README.md
│   ├── index.html
│   ├── RFC-0001.md      Cyclone là gì
│   ├── RFC-0002.md      Wire Format Specification
│   └── RFC-0003.md      Conformance
│
└── en/                  tiếng Anh — bản dịch
    ├── RFC-0001.md
    ├── RFC-0002.md
    └── RFC-0003.md
```

## Giấy phép

Toàn bộ repo này theo [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) — xem [`LICENSE`](../LICENSE).

Bạn được tự do sao chép, dịch, trích dẫn và tạo tác phẩm phái sinh từ specification, kể cả cho mục đích thương mại, miễn là **ghi nhận tác giả**.

Đây là **specification, không phải phần mềm**. Implementation của bạn là một tác phẩm riêng biệt, không phải tác phẩm phái sinh của specification — bạn được toàn quyền chọn giấy phép cho nó.

## Bản quyền

Copyright © 2026 Ha Duy Thang
