# Giao thức Cyclone

[Tiếng Anh](../README.md) · **Tiếng Việt**

Cyclone định nghĩa cách dữ liệu được ghi dưới dạng các byte.

Chỉ vậy thôi.

---

## Giải thích nhanh trong 30 giây

Bạn có một đối tượng `Player`:

```
Hp   = 100
Name = "Alice"
```

Cyclone quy định rằng nó sẽ trở thành chính xác chuỗi byte sau:

```
64 00 00 00   05 00 00 00   41 6C 69 63 65
Hp = 100    Độ dài = 5      A  l  i  c  e
```

13 byte. Không tên trường, không dấu ngoặc, không thẻ (tag) - chỉ thuần túy là dữ liệu.

Điểm mấu chốt: Go, C#, Rust và C đều phải tạo ra chính xác 13 byte này. Không chỉ là "tương đương", mà phải giống hệt nhau đến từng byte.

---

## Tại sao lại cần điều này?

Với JSON hoặc Protobuf, cùng một dữ liệu có thể tạo ra các chuỗi byte khác nhau tùy thuộc vào ngôn ngữ, thư viện hoặc phiên bản. Thông thường, điều đó không thành vấn đề. Nhưng nó sẽ gây lỗi khi:

- Bạn ký hoặc băm (hash) dữ liệu (payload) - chữ ký dựa trên các byte; nếu byte thay đổi, chữ ký sẽ không còn hợp lệ ngay cả khi dữ liệu vẫn giữ nguyên.
- Bạn chạy mô phỏng phát lại (replay) hoặc đồng bộ từng bước (lockstep) - hai máy tính phải tạo ra cùng một kết quả từ cùng một chuỗi đầu vào.
- Bạn muốn xác minh tính tương thích của SDK - cách duy nhất là so sánh các byte.

Cyclone giải quyết vấn đề này bằng cách loại bỏ các lựa chọn: bất cứ khi nào bộ mã hóa (encoder) có quyền lựa chọn, các bộ mã hóa khác nhau chắc chắn sẽ chọn theo cách khác nhau.

---

## Cyclone không phải là gì

```
- Networking framework    - RPC
- Game engine             - Transport (TCP/UDP/QUIC)
- Encryption              - Compression
```

Những thứ đó được xây dựng dựa trên Cyclone. Cyclone hoạt động nghiêm ngặt tại điểm dữ liệu thay đổi hình thức:

```
Model  →  Bytes
Bytes  →  Model
```

---

## Cách thức hoạt động

Chỉ cần hiểu bốn quy tắc là đủ để nắm bắt toàn bộ hệ thống:

1. Các số có kích thước cố định, định dạng little-endian. Một `UInt32` luôn chiếm 4 byte, ngay cả khi giá trị là 0. Không sử dụng varint (số nguyên kích thước biến đổi).

2. Chuỗi và mảng có độ dài xác định ở đầu. `[độ dài 4 byte][dữ liệu]`. Đối với chuỗi, độ dài biểu thị số byte UTF-8, không phải số lượng ký tự.

3. Các mô hình (model) bao gồm các trường nằm liền kề nhau. Không có phần tiêu đề (header), không có phần đệm (padding), không có ký tự phân cách. Bộ giải (decoder) mã xác định trường hiện tại dựa trên vị trí con trỏ, không phải thông qua các thẻ (tag).

4. Không có siêu dữ liệu (metadata). Luồng byte không tự mô tả chính nó. Bên nhận bắt buộc phải biết trước chính xác thứ tự và kiểu dữ liệu của từng trường.

Hệ quả của quy tắc 3 và 4 - cần hiểu rõ trước khi sử dụng:

```
Thay đổi tên trường      →  không thay đổi byte          (an toàn)
Thay đổi thứ tự trường   →  thay đổi toàn bộ luồng byte   (gây lỗi tương thích)
Định nghĩa sai           →  thường không báo lỗi, tạo ra dữ liệu sai mà không có cảnh báo
```

---

## Cyclone không bắt buộc sử dụng một IDL cụ thể

Nguồn thông tin chuẩn (source of truth) chính là **Schema**-một tập hợp bao gồm tên kiểu dữ liệu, tên trường, kiểu Cyclone và thứ tự các trường. Luồng byte được tạo ra từ Schema, và *chỉ* từ Schema đó mà thôi.

Tuy nhiên, Cyclone không quy định *cách* bạn phải viết Schema đó:

```
Nhiều cách thể hiện   →   một Schema   →   duy nhất một chuỗi byte
```

Không bắt buộc phải có tệp `.cyclone`. Không bắt buộc phải dùng trình biên dịch IDL nào cụ thể. Miễn là cả hai bên cùng tạo ra một Schema giống nhau, thì kết quả byte thu được sẽ hoàn toàn trùng khớp.

Bản triển khai tham chiếu chọn cách thể hiện schema bằng các chú thích (annotation) trực tiếp trên các mô hình hiện có - ví dụ trong C#:

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

Nó không cần phải hiểu các từ khóa `public`, `partial`, `class`, `uint`, hay việc `PlayerInfo` là class hay struct. Chỉ cần trích xuất ba yếu tố cụ thể để xây dựng Schema:

```
Tên kiểu (Type Name)        →  Player
Tên trường (Field Name)     →  Hp
Kiểu Cyclone (Cyclone Type) →  UInt32
```

Kiểu Cyclone được xác định từ chú thích (annotation) chứ không phải được suy luận từ hệ thống kiểu của ngôn ngữ chủ. Việc đặt `[Network(UInt32)]` lên một trường không tương thích sẽ dẫn đến lỗi khai báo-Cyclone không tự đưa ra các giả định thay cho bạn.

Kết quả là: việc bổ sung hỗ trợ cho một ngôn ngữ mới trở thành một tác vụ đơn giản, gọn nhẹ. Thành phần frontend chỉ cần đọc chú thích, tên kiểu và tên trường; nó không đòi hỏi phân tích ngữ nghĩa hay sự am hiểu về hệ thống kiểu của ngôn ngữ đích.

Một cách triển khai khác hoàn toàn có thể chọn hướng tiếp cận khác - ví dụ như sử dụng tệp schema riêng, macro, hoặc tự viết mã codec. Chỉ có một tiêu chuẩn duy nhất để đánh giá: liệu các byte dữ liệu được tạo ra có chính xác hay không.

---
## Khi nào KHÔNG nên sử dụng

Cần làm rõ rằng:

- Lược đồ (schema) thường xuyên thay đổi → mọi sửa đổi phải tránh làm thay đổi thứ tự trường hoặc xóa các trường nằm ở giữa; các trường chỉ được phép thêm vào hoặc loại bỏ ở vị trí cuối cùng.
- Có nhiều trường "tùy chọn" (optional) → Phiên bản v1 thiếu hỗ trợ cho kiểu Optional/Nullable.

Nếu bất kỳ trường hợp nào trên đây áp dụng cho bạn, Protobuf sẽ là lựa chọn tốt hơn.

---

## Cam kết của Cyclone

```
Cùng lược đồ, cùng giá trị

↓

Tất cả các bản triển khai (implementation) PHẢI tạo ra các chuỗi byte giống hệt nhau.
```

Nếu có sự khác biệt, đó là lỗi triển khai, không phải lỗi giao thức. Các lỗi như vậy có thể được phát hiện bằng cách so sánh các mảng byte - hãy tham khảo tài liệu về sự tuân thủ (Conformance).

---

## Tài liệu

Hãy đọc theo thứ tự sau:

| Tài liệu | Giải đáp câu hỏi |
|----------|-----------------|
| [RFC-0001 - Cyclone là gì](RFC-0001.md) | Nó giải quyết vấn đề gì, tại sao nên chọn nó và khi nào không nên sử dụng |
| [RFC-0002 - Định dạng truyền tải (Wire Format)](RFC-0002.md) | Cấu trúc byte trông như thế nào |
| [RFC-0003 - Sự tuân thủ (Conformance)](RFC-0003.md) | Làm thế nào để xác minh bản triển khai của tôi có chính xác không (các bộ dữ liệu kiểm thử - test vectors) | Bản dịch tiếng Anh: [`en/RFC-0001.md`](../en/RFC-0001.md) · [`en/RFC-0002.md`](../en/RFC-0002.md) · [`en/RFC-0003.md`](../en/RFC-0003.md)

Trang chủ: [cyclone-protocol.github.io/cyclone](https://cyclone-protocol.github.io/cyclone/) - mã nguồn tại [`index.html`](../index.html) (tiếng Anh) và [`vi/index.html`](index.html) (tiếng Việt)

---

## Đóng góp

Cyclone là một bản đặc tả kỹ thuật (specification), không phải là một thư viện. Cần có các bản triển khai cho Rust, Go, C#/Unity, C/embedded và Zig/C++.

Tiêu chí duy nhất để đạt chuẩn Cyclone Compatible (Tương thích Cyclone):

```
Chạy các bộ dữ liệu kiểm thử (test vectors) trong RFC-0003

↓

Đạt 100%
```

Không có khái niệm "98%". Chỉ cần một bộ dữ liệu kiểm thử thất bại cũng đồng nghĩa với việc có một phần dữ liệu mà bản triển khai của bạn và một bản triển khai khác không thống nhất với nhau.

Hãy xem [bảng trạng thái hệ sinh thái](https://cyclone-protocol.github.io/cyclone/#implementations) để biết những ngôn ngữ nào hiện chưa có bản triển khai.

### Các bản dịch

Tiếng Việt là ngôn ngữ gốc; tiếng Anh là bản dịch.

---

## Cấu trúc kho lưu trữ (Repo)

```
.
├── README.md            tài liệu này (tiếng Anh)
├── LICENSE              CC BY 4.0
│
├── vi/                  Tiếng Việt - phiên bản gốc
│   ├── README.md
│   ├── RFC-0001.md      Cyclone là gì
│   ├── RFC-0002.md      Đặc tả định dạng truyền tải (Wire Format)
│   └── RFC-0003.md      Sự tuân thủ (Conformance)
│
└── en/                  Tiếng Anh - phiên bản dịch
├── RFC-0001.md
├── RFC-0002.md
└── RFC-0003.md
```

## Giấy phép

Toàn bộ kho lưu trữ này được cấp phép theo [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) - xem tệp [`LICENSE`](../LICENSE).

Bạn được tự do sao chép, dịch, trích dẫn và tạo các tác phẩm phái sinh dựa trên đặc tả này, bao gồm cả mục đích thương mại, với điều kiện bạn ghi nhận nguồn/tác giả một cách phù hợp.

Đây là một bản đặc tả kỹ thuật, không phải phần mềm. Bản triển khai của bạn là một tác phẩm độc lập, không phải là tác phẩm phái sinh của bản đặc tả này - bạn có quyền tự do lựa chọn giấy phép cho nó.

## Bản quyền

Copyright © 2026 Ha Duy Thang