# 19. System Design — Interview Mode

> Dùng file này **trước** khi xem reference answer trong `docs/12-system-design.md`.

## Interview protocol

Thời lượng gợi ý: **35–45 phút**.

### Phase 1 — Clarify (5 phút)

Ứng viên phải hỏi:

- Ai là user?
- DAU/MAU và traffic peak?
- Read/write ratio?
- SEO có quan trọng không?
- Realtime requirement là bao nhiêu?
- Consistency requirement?
- Browser/device constraints?
- Team size và ownership?

### Phase 2 — High-level design (10 phút)

Vẽ:

```text
User
 ↓
Web / Mobile Web
 ↓
Frontend Application
 ↓
BFF / API
 ↓
Backend services
 ↓
Data / external services
```

Không cần chọn framework ngay. Hãy giải quyết requirements trước.

### Phase 3 — Deep dive (15 phút)

Interviewer chọn 2–3 chủ đề:

- state ownership
- caching
- rendering strategy
- realtime
- large list
- error handling
- authentication
- observability
- deployment
- SEO

### Phase 4 — Trade-offs (5 phút)

Ứng viên phải trả lời:

> Nếu traffic tăng 10x thì phần nào trở thành bottleneck đầu tiên?

> Nếu team giảm một nửa thì kiến trúc nào nên đơn giản hóa?

> Nếu SEO không còn requirement thì bạn bỏ gì?

### Phase 5 — Rollout (5 phút)

Hỏi:

- Migration strategy?
- Feature flag?
- Backward compatibility?
- Monitoring?
- Rollback?

---

# Case 01 — E-commerce

## Prompt

Thiết kế frontend cho e-commerce có:

- 100k SKU
- Search/filter/sort
- Product detail
- Cart
- Checkout
- Payment
- Order tracking
- SEO
- Analytics

Không đưa architecture cho candidate trước.

## Follow-up

1. Search có cần SSR không?
2. Filter state nên nằm ở đâu?
3. Cart là server state hay client state?
4. Làm sao tránh refetch toàn bộ product list khi filter thay đổi?
5. Product detail có cần cache?
6. Checkout retry payment thế nào?
7. Nếu catalog tăng 10x thì frontend thay đổi gì?

## Scoring signals

**Senior:** phân biệt URL/server/client/local state, caching, pagination, SEO và failure handling.

**Staff:** giải thích ownership, boundary, observability, migration và business trade-offs.

---

# Case 02 — Realtime Chat

## Prompt

Thiết kế chat application:

- 1:1 chat
- Group chat
- Online/offline status
- Message history
- Typing indicator
- Read receipt
- Reconnect
- Multi-tab

## Follow-up

1. WebSocket reconnect thế nào?
2. Message ordering xử lý ra sao?
3. Duplicate message khi reconnect?
4. Offline user nhận message thế nào?
5. 10k messages trong conversation render ra sao?
6. Multi-tab có mở nhiều socket không?
7. Nếu server gửi burst 2,000 events/s thì UI làm gì?

## Scoring signals

Candidate tốt phải phân biệt **transport reliability** với **UI rendering performance** và không giải quyết mọi vấn đề bằng WebSocket.

---

# Case 03 — Big Data Dashboard

## Prompt

Thiết kế dashboard operational:

- 20 widgets
- realtime metrics
- filter theo thời gian
- bảng hàng chục nghìn records
- export report
- role-based access

## Follow-up

1. Widget nào query riêng, widget nào share query?
2. Cache boundary ở đâu?
3. Virtualization giải quyết vấn đề nào?
4. Export chạy ở browser hay backend?
5. User đổi filter liên tục thì cancellation/deduplication thế nào?
6. RBAC enforce ở frontend hay backend?
7. Làm sao đo dashboard có thực sự nhanh hơn?

## Expected thinking

Không bắt đầu bằng `React.memo` hoặc virtualization. Trước tiên xác định:

```text
Data volume
   ↓
Network cost
   ↓
Parsing cost
   ↓
State update frequency
   ↓
Render cost
   ↓
DOM cost
```

Sau đó mới chọn optimization phù hợp.

---

# Interview scorecard

| Dimension | 1 | 2 | 3 | 4 |
|---|---|---|---|---|
| Requirement | Không clarify | Hỏi vài câu | Clarify có hệ thống | Xác định cả hidden constraints |
| Architecture | Fragmented | Có happy path | Có boundaries/trade-offs | Có evolution/migration strategy |
| Frontend | Tool-centric | Biết patterns | Chọn theo requirement | Giải thích ownership + failure modes |
| Performance | Đoán | Biết vài technique | Profile rồi optimize | Tối ưu theo bottleneck + RUM |
| Reliability | Ít đề cập | Có retry/error | Có fallback/rollback | Có observability + incident strategy |
| Communication | Lan man | Có cấu trúc | Rõ ràng | Dẫn dắt discussion và nêu assumption |

**Senior target:** đa số tiêu chí đạt 3.

**Staff signal:** nhiều tiêu chí đạt 4 mà không over-engineer.
