# Must-Have Docs — Senior Frontend Interview (Kỹ Thuật)

> **Mục tiêu:** Thay thế `docs/25-25-khai-niem-tinh-hoa.md` cũ (dùng ẩn dụ `balo`, `ngã tư`, tự đặt tên) bằng bộ tài liệu **giữ nguyên thuật ngữ chuyên môn** như khi phỏng vấn thật hỏi. Mỗi file là một khái niệm/kỹ thuật/vấn đề bắt buộc, viết theo dạng kỹ thuật nhưng giải thích chi tiết để dễ hiểu.

## Nguyên tắc viết (bắt buộc cho mọi file)

1. **Giữ thuật ngữ gốc:** `Closure`, `LexicalEnvironment`, `TDZ`, `Hoisting`, `Prototype Chain`, `Event Loop`, `Microtask`, `Fiber`, `Reconciliation`, `Hydration`, `CORS`, `SOP`, `LCP/INP/CLS`, v.v. — không đổi thành tên tự đặt.
2. **Không dùng:** `nôm na`, `gom từ câu nào`, ẩn dụ balo/ngã tư/dây chuyền sơn. Thay bằng định nghĩa spec + cơ chế.
3. **Cấu trúc mỗi file (6 phần, ngắn gọn):**
   - **Định nghĩa chính xác (1-2 câu):** Dùng thuật ngữ chuẩn, trích spec nếu cần (ECMA-262, HTML Living Standard, React docs).
   - **Cơ chế hoạt động:** Engine / Runtime làm gì, thứ tự, data structure.
   - **Ví dụ tối thiểu runnable:** 1-2 snippet JS/TSX, có comment kết quả; tránh ví dụ dài.
   - **Trade-off / Khi nào KHÔNG dùng (When not to use):**
   - **Failure modes & cách đo / debug:** Lỗi thường gặp + công cụ (Chrome DevTools, `performance`, `web-vitals`, `madge`, `wdyr`...).
   - **Câu hỏi tự kiểm tra (3 câu):** Dạng phỏng vấn, có đáp án ngắn 30s ở cuối file (collapsed hoặc footnote).
4. **Tone:** Kỹ thuật, chính xác, senior. Giải thích chi tiết nhưng không lan man. Dùng bảng khi so sánh.
5. **Link chéo:** Mỗi file link tới docs gốc chi tiết `docs/01-...` để đào sâu.

## Lộ trình 7 ngày (đề xuất)

| Ngày | Thư mục | Trọng tâm |
|------|---------|-----------|
| 1 | `01-javascript` | Scope/TDZ, Closure, `this`, Prototype, Event Loop |
| 2 | `02-typescript` | Type system, narrowing, generics, conditional/infer |
| 3 | `03-react-core` | Render/reconcile/key, hooks, memo, context |
| 4 | `04-react-advanced` | Fiber/concurrent, Suspense/EB, RSC/SSR/hydration, cache |
| 5 | `05-browser-performance` + `06-network-security` | Pipeline, Web Vitals, bundle, CORS/HTTP, REST |
| 6 | `07-architecture` | FSD, design system, data sync, pagination |
| 7 | `08-state-management` + `09-testing` | Server/client/URL state, TanStack Query, store, testing |

## Cấu trúc thư mục

```
must-have-docs/
├── README.md
├── 01-javascript/              # 10 files — JS nền
├── 02-typescript/              # 7 files — TS
├── 03-react-core/              # 6 files — React core
├── 04-react-advanced/          # 7 files — React 18/19, RSC, SSR
├── 05-browser-performance/     # 6 files — Rendering, Vitals, bundle
├── 06-network-security/        # 5 files — CORS, HTTP, REST, auth
├── 07-architecture/            # 6 files — FSD, data sync
├── 08-state-management/        # 4 files — Server/client/URL, stores
└── 09-testing/                 # 3 files — Pyramid, RTL/MSW, debugging
```

Tổng: ~54 files, phủ 185/199 câu (trừ Next.js ch.11 theo yêu cầu cũ). Mỗi file ~250-400 dòng, đọc 5-7 phút.

## Mapping với 199 câu gốc

| must-have-docs | Phủ docs gốc |
|----------------|--------------|
| `01-javascript` | `docs/01` câu 1-18 |
| `02-typescript` | `docs/01` câu 19-30 |
| `03-react-core` | `docs/02` câu 31-50 |
| `04-react-advanced` | `docs/03` câu 51-68 |
| `05-browser-performance` | `docs/05` + `docs/06` |
| `06-network-security` | `docs/08` + `docs/07` |
| `07-architecture` | `docs/04` + `docs/12` |
| `08-state-management` | `docs/09` + TanStack |
| `09-testing` | `docs/10` |

## Quy ước đặt tên file

- `kebab-case`, prefix số `01-`, tên là thuật ngữ: `02-closure-lexical-environment.md`, `04-event-loop-microtask-macrotask.md`
- Mỗi file bắt đầu bằng `# <Thuật ngữ> — <Mô tả 1 dòng>` và có `Tags: #js #closure` để search.

## Cách dùng khi ôn

- **Bận (3 ngày):** `01-javascript` + `03-react-core` + `04-react-advanced/03-rsc-ssr-ssg-isr`
- **Đủ (7 ngày):** Đi theo lộ trình trên, mỗi file tự trả lời 3 câu cuối không nhìn đáp án, rồi code lại ví dụ.
- **Trước phỏng vấn 30p:** Đọc lại phần **Failure modes** + **Trade-off** của mỗi file — đó là nơi interviewer xoáy.

---
*Tạo bởi workflow `must-have-docs`. Nguồn chi tiết: `docs/01-14`. Giữ nguyên thuật ngữ để khi bị hỏi “Closure là gì? TDZ khác hoisting thế nào?” bạn trả lời đúng từ khóa.*
