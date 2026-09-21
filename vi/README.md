# Giao thức Fomoxa

[Tiếng Anh](../README.md) · Tiếng Việt

Fomoxa định nghĩa cách dữ liệu được ghi thành byte và không định nghĩa gì khác.

## Giải thích trong 30 giây

Xét một đối tượng `Player`:

```
Hp   = 100
Name = "Alice"
```

Fomoxa quy định nó phải thành đúng chuỗi byte sau:

```
64 00 00 00   05 00 00 00   41 6C 69 63 65
Hp = 100      Độ dài = 5    A  l  i  c  e
```

Kết quả là 13 byte dữ liệu, không có tên field, dấu ngoặc hay tag.

Go, C#, Rust và C đều phải sinh đúng 13 byte này. Kết quả tương đương là chưa đủ; byte phải giống hệt nhau.

## Tại sao cần điều này?

Với JSON hoặc Protobuf, cùng một dữ liệu có thể ra nhiều chuỗi byte khác nhau. Khác ngôn ngữ, khác thư viện, khác phiên bản là khác byte. Phần lớn trường hợp điều đó vô hại. Nó gây lỗi trong các trường hợp sau:

- Ký hoặc hash payload. Chữ ký gắn với byte, nên byte đổi thì chữ ký hỏng dù dữ liệu logic không đổi.
- Chạy replay hoặc lockstep. Hai máy phải ra cùng kết quả từ cùng chuỗi đầu vào.
- Xác minh hai SDK tương thích. Cách duy nhất là so byte.

Fomoxa xử lý bằng cách loại bỏ mọi lựa chọn của encoder. Ở đâu encoder được quyền chọn, ở đó hai encoder sẽ chọn khác nhau.

## Fomoxa không phải là gì

```
- Networking framework    - RPC
- Game engine             - Transport (TCP/UDP/QUIC)
- Encryption              - Compression
```

Những thứ đó nằm trên Fomoxa. Fomoxa đứng đúng tại chỗ dữ liệu đổi hình dạng:

```
Model  →  Bytes
Bytes  →  Model
```

## Cách thức hoạt động

Toàn bộ hệ thống gồm bốn quy tắc:

1. Số có kích thước cố định, little-endian. Một `UInt32` luôn chiếm 4 byte, kể cả khi giá trị bằng 0. Không dùng varint.

2. String và Array có độ dài đứng trước: `[độ dài 4 byte][dữ liệu]`. Với String, độ dài là số byte UTF-8, không phải số ký tự.

3. Model là các field nằm liền nhau, không có header, padding hay dấu phân cách. Decoder xác định field bằng vị trí con trỏ, không bằng tag.

4. Không có metadata. Luồng byte không tự mô tả, nên bên nhận phải biết trước thứ tự và kiểu của từng field.

Quy tắc 3 và 4 dẫn tới các hệ quả sau, cần nắm trước khi dùng:

```
Đổi tên field      →  byte không đổi              (an toàn)
Đổi thứ tự field   →  đổi toàn bộ luồng byte      (breaking change)
Định nghĩa sai     →  không báo lỗi, dữ liệu sai  (im lặng)
```

## Fomoxa không ép một IDL cụ thể

Nguồn sự thật là schema: tên kiểu, tên field, kiểu Fomoxa, thứ tự field. Luồng byte sinh ra từ schema, và *chỉ* từ schema.

Fomoxa không quy định *cách* viết schema:

```
Nhiều cách thể hiện   →   một schema   →   một chuỗi byte
```

Không bắt buộc file `.fomoxa` hay một trình biên dịch IDL cụ thể. Hai bên dựng ra cùng một schema thì byte trùng khớp.

Reference implementation thể hiện schema bằng annotation đặt trên kiểu có sẵn. Ví dụ trong C#:

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

Implementation này không cần hiểu `public`, `partial`, `class` hay `uint`, cũng không cần biết `PlayerInfo` là class hay struct. Nó chỉ rút ba thứ để dựng schema:

```
Tên kiểu (Type Name)        →  Player
Tên field (Field Name)      →  Hp
Kiểu Fomoxa (Fomoxa Type)   →  UInt32
```

Kiểu Fomoxa lấy từ annotation, không suy ra từ hệ thống kiểu của ngôn ngữ chủ. Gắn `[Network(UInt32)]` lên một field không tương thích là lỗi khai báo; Fomoxa không tự đoán.

Nhờ đó, thêm một ngôn ngữ mới tốn ít công sức. Frontend chỉ đọc annotation, tên kiểu và tên field, không cần phân tích ngữ nghĩa hay hiểu hệ thống kiểu của ngôn ngữ đích.

Một implementation khác có thể đi đường khác: file schema riêng, macro, hoặc codec viết tay. Thước đo duy nhất là byte sinh ra có đúng hay không.

## Khi nào KHÔNG nên dùng

- Schema đổi liên tục. Mọi sửa đổi phải giữ nguyên thứ tự field và không xoá field ở giữa; field chỉ được thêm hoặc bớt ở cuối.
- Có nhiều field "optional". v1.1 không có Optional/Nullable.

Nếu rơi vào một trong hai trường hợp trên, hãy dùng Protobuf.

## Cam kết của Fomoxa

```
Cùng schema, cùng giá trị

↓

Mọi implementation PHẢI sinh ra byte giống hệt nhau.
```

Lệch nhau là lỗi implementation, không phải lỗi giao thức, và phát hiện được bằng cách so hai mảng byte. Xem RFC-0003.

## Tài liệu

Đọc theo thứ tự:

| Tài liệu | Giải đáp câu hỏi |
|----------|-----------------|
| [RFC-0001 - Fomoxa là gì](RFC-0001.md) | Nó giải quyết vấn đề gì, tại sao chọn nó, khi nào không nên dùng |
| [RFC-0002 - Wire Format](RFC-0002.md) | Cấu trúc byte trông như thế nào |
| [RFC-0003 - Conformance](RFC-0003.md) | Làm sao biết implementation của tôi có đúng không (test vector) |

Bản dịch tiếng Anh: [`en/RFC-0001.md`](../en/RFC-0001.md) · [`en/RFC-0002.md`](../en/RFC-0002.md) · [`en/RFC-0003.md`](../en/RFC-0003.md)

Trang chủ: [fomoxa.github.io](https://fomoxa.github.io/). Mã nguồn tại [`index.html`](../index.html) (tiếng Anh) và [`vi/index.html`](index.html) (tiếng Việt).

## Đóng góp

Fomoxa là một đặc tả, không phải thư viện. Cần implementation cho Rust, Go, C#/Unity, C/embedded và Zig/C++.

Tiêu chí duy nhất để đạt chuẩn Fomoxa Compatible:

```
Chạy test vector trong RFC-0003

↓

Đạt 100%
```

Mức 98% không được tính là đạt. Một test vector trượt nghĩa là tồn tại dữ liệu mà implementation của bạn và implementation khác hiểu khác nhau.

Xem [bảng trạng thái hệ sinh thái](https://fomoxa.github.io/#implementations) để biết ngôn ngữ nào còn trống.

### Các bản dịch

Tiếng Việt là ngôn ngữ gốc. Tiếng Anh là bản dịch.

## Cấu trúc repo

```
.
├── README.md            bản tiếng Anh
├── LICENSE              CC BY 4.0
│
├── vi/                  Tiếng Việt - bản gốc
│   ├── README.md        tài liệu này
│   ├── RFC-0001.md      Fomoxa là gì
│   ├── RFC-0002.md      Wire Format
│   └── RFC-0003.md      Conformance
│
└── en/                  Tiếng Anh - bản dịch
    ├── RFC-0001.md
    ├── RFC-0002.md
    └── RFC-0003.md
```

## Giấy phép

Toàn bộ repo này cấp phép theo [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/). Xem file [`LICENSE`](../LICENSE).

Bạn được tự do sao chép, dịch, trích dẫn và tạo tác phẩm phái sinh, kể cả cho mục đích thương mại, với điều kiện ghi nhận nguồn đúng cách.

Đây là một đặc tả, không phải phần mềm. Implementation của bạn là tác phẩm độc lập, không phải tác phẩm phái sinh của đặc tả này, và bạn tự chọn giấy phép cho nó.

## Bản quyền

Copyright © 2026 Ha Duy Thang
