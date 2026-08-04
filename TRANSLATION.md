# Translation Workflow

Cyclone is written in two languages. They are not equal partners.

```
vi/   Vietnamese   source of record   - the author writes here
en/   English      translation        - derived from vi/
```

If the two versions ever disagree on a normative statement, **the Vietnamese text wins**, and the English text is a bug to be fixed.

---

## The process

```
1. The author writes or edits the RFC in Vietnamese      →  vi/RFC-000X.md
2. Claude translates it into English                     →  en/RFC-000X.md
3. The author reviews technical terminology only
```

Step 3 is deliberately narrow. The author is not proofreading English prose - the author is checking that the terms that carry protocol meaning came across intact.

## What step 3 checks

| Must be preserved exactly | Notes |
|---------------------------|-------|
| RFC 2119 keywords | `MUST`, `MUST NOT`, `SHOULD`, `MAY` - never softened, never added, never dropped |
| Type names | `UInt32`, `f32`, `bool`, `String`, `Bytes`, `Array<T>`, `Enum` |
| Byte values and hex dumps | Never re-formatted, never re-spaced, never re-ordered |
| Vector IDs | `P-032`, `F-031`, `N-023`, `T-010`, `X-003`… |
| Section numbers and cross-references | `§2.3`, `§10.1`, RFC numbers |
| Terms of art | wire format, byte stream, endianness, bit pattern, canonical, normative, round-trip, conformance, schema evolution, length prefix, declaration order |
| Numbers | sizes, counts, limits, byte totals |

Anything outside that table - sentence rhythm, paragraph breaks, word choice in explanatory prose - is the translator's call and does not need review.

## Rules for the translation

```
1. Translate meaning, not word order. English spec prose has its own idiom.
2. Never translate a term of art into a descriptive phrase.
   "bit pattern" stays "bit pattern", not "the pattern of bits".
3. Never translate content inside code blocks, tables of bytes, or vector IDs.
4. Never add a claim the Vietnamese text does not make - including helpful
   clarifications. A clarification worth having belongs in vi/ first.
5. Never drop a hedge or a caveat. "thường KHÔNG báo lỗi" is "usually raises
   NO error", not "raises no error".
6. Keep the document structure identical: same headings, same numbering,
   same table columns, same order.
```

Rule 4 is the one that matters most. The English file is a mirror, not an improved edition. If the translation surfaces a gap or an ambiguity, fix `vi/` and re-translate - do not patch it in English only.

## Every English file carries a pointer

Each file in `en/` opens with:

```markdown
> Translated from the Vietnamese source of record: [`vi/RFC-000X.md`](../vi/RFC-000X.md).
> If the two disagree, the Vietnamese text wins.
```

This is not decoration. It tells a reader who finds an inconsistency which file to trust and which one to file an issue against.

## The website

The landing page follows the same rule:

```
index.html       English   (GitHub Pages entry point)
vi/index.html    Vietnamese
```

Both carry a `EN / VI` switch in the top bar. The English page is the entry point because that is where GitHub Pages serves `/`; that is a hosting detail, not a statement about which language is authoritative.

---

# Quy trình dịch

Cyclone được viết bằng hai ngôn ngữ. Hai bản không ngang hàng nhau.

```
vi/   Tiếng Việt   bản gốc      - tác giả viết ở đây
en/   Tiếng Anh    bản dịch     - dịch từ vi/
```

Nếu hai bản mâu thuẫn ở một phát biểu normative, **bản tiếng Việt đúng**, và bản tiếng Anh là lỗi cần sửa.

## Quy trình

```
1. Tác giả viết / sửa RFC bằng tiếng Việt   →  vi/RFC-000X.md
2. Claude dịch sang tiếng Anh               →  en/RFC-000X.md
3. Tác giả chỉ review thuật ngữ kỹ thuật
```

Bước 3 cố ý hẹp. Tác giả không soát văn phong tiếng Anh - chỉ kiểm tra những thuật ngữ mang nghĩa protocol có sang đúng hay không: từ khoá RFC 2119, tên kiểu, giá trị byte, ID test vector, số hiệu mục, và các con số.

Nếu bản dịch làm lộ ra một chỗ mơ hồ, sửa ở `vi/` rồi dịch lại - không vá riêng bên tiếng Anh.
