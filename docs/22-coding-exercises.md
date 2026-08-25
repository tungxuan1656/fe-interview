# 22. Senior Coding Exercises

> Coding round không chỉ kiểm tra thuật toán. Với Senior Frontend, interviewer thường muốn thấy **API design, edge cases, complexity, browser constraints, testability và khả năng giải thích trade-off**.

## Cách luyện

Mỗi bài theo 4 bước:

1. Nói rõ assumption và API trước khi code.
2. Viết solution đơn giản, đúng trước.
3. Phân tích complexity + edge cases.
4. Đề xuất production hardening và test cases.

Không xem reference trước khi hoàn thành phần **What interviewer looks for**.

---

## Exercise 01 — Debounce có `cancel` và `flush`

### Prompt

Implement:

```ts
function debounce<T extends (...args: any[]) => any>(
  fn: T,
  wait: number,
): T & { cancel(): void; flush(): void }
```

Yêu cầu:

- Chỉ chạy sau khi không có call mới trong `wait`.
- `cancel()` hủy invocation đang chờ.
- `flush()` chạy invocation đang chờ ngay lập tức.
- Giữ đúng `this` và arguments.

### Follow-up

- Leading/trailing mode?
- Nếu `fn` throw thì state nội bộ có bị kẹt không?
- Dùng trong search UI có cần AbortController không?

### Interviewer looks for

Closure, timer lifecycle, API semantics, edge cases và khả năng phân biệt debounce với request cancellation.

---

## Exercise 02 — Concurrent request limiter

### Prompt

Viết utility nhận danh sách async tasks và giới hạn tối đa `N` task chạy đồng thời.

```ts
mapWithConcurrency(tasks, 3)
```

### Follow-up

- Preserve input order?
- Một task reject thì có dừng toàn bộ không?
- Có cancellation không?
- Retry có làm vượt concurrency limit không?

### Interviewer looks for

Queue, scheduling, failure semantics và resource control.

---

## Exercise 03 — LRU cache

### Prompt

Implement generic `LRUCache<K, V>` với `get`, `set`, `has`, `delete` và capacity cố định.

### Follow-up

- Target `O(1)` cho `get/set`.
- TTL có nên nằm trong cùng abstraction không?
- Cache server state khác memoization thế nào?

### Interviewer looks for

Data structure choice và khả năng tránh over-engineering.

---

## Exercise 04 — Request deduplication

### Prompt

Nếu 10 component cùng request `/api/user/123` trong cùng thời điểm, chỉ gửi một request và share Promise cho các consumers.

### Follow-up

- Promise reject thì cache entry xử lý thế nào?
- Request lifetime kết thúc lúc nào?
- Nếu một consumer muốn cancel thì có được abort shared request không?

### Interviewer looks for

In-flight cache, ownership và cancellation semantics.

---

## Exercise 05 — Stale response protection

### Prompt

Implement search request manager sao cho response của request cũ không thể overwrite kết quả mới.

### Follow-up

So sánh sequence number, AbortController, query library và server-side request IDs.

### Interviewer looks for

Hiểu rằng cancellation và stale-response protection là hai vấn đề liên quan nhưng không đồng nhất.

---

## Exercise 06 — Virtualized list

### Prompt

Thiết kế algorithm render một list 1,000,000 rows nhưng chỉ mount phần tử nằm trong viewport + overscan.

### Follow-up

Variable row height? Scroll position restore? Keyboard navigation? Accessibility?

### Interviewer looks for

Measurement, overscan, rendering cost và UX constraints.

---

## Exercise 07 — Event emitter an toàn

### Prompt

Implement `on`, `off`, `once`, `emit` với TypeScript typed events.

### Follow-up

Listener tự unsubscribe trong lúc emit? Listener throw exception? Memory leak detection?

### Interviewer looks for

Mutation trong iteration, lifecycle và type-safe API design.

---

## Exercise 08 — Deep equality

### Prompt

Implement `isEqual(a, b)` cho object/array/primitive.

### Follow-up

Circular reference? Date/Map/Set? NaN và `-0`? Có nên dùng JSON.stringify không?

### Interviewer looks for

Phạm vi bài toán trước khi code và khả năng nói rõ giới hạn implementation.

---

## Exercise 09 — Retry with backoff

### Prompt

Implement retry cho async operation với exponential backoff và maximum attempts.

### Follow-up

Retry mọi HTTP status có đúng không? Jitter để làm gì? POST có retry an toàn không? Timeout kết hợp retry thế nào?

### Interviewer looks for

Idempotency, retry storms, timeout và backoff.

---

## Exercise 10 — Build a tiny state store

### Prompt

Implement minimal store:

```ts
const store = createStore(initialState)
store.getState()
store.setState(...)
store.subscribe(listener)
```

### Follow-up

Selector subscription? Prevent unnecessary renders? Batched updates? Persistence? Server state có nên đưa vào store không?

### Interviewer looks for

State ownership, subscriptions và rendering implications hơn là syntax.

---

## Exercise 11 — Priority task scheduler

### Prompt

Thiết kế scheduler có `high`, `normal`, `low` priority và không block main thread quá lâu.

### Follow-up

`requestIdleCallback` có đủ không? Yielding giữa tasks? Khi nào `startTransition` phù hợp hơn?

### Interviewer looks for

Browser main-thread model và distinction giữa application scheduler với React scheduler.

---

## Exercise 12 — Transform large API response

### Prompt

API trả 100,000 records. Transform dữ liệu để hiển thị nhưng không được làm UI freeze.

### Follow-up

Chunk work? Web Worker? Streaming? Memoization? Có cần transform toàn bộ data không?

### Interviewer looks for

Tối ưu pipeline thay vì chỉ tối ưu một function.

---

## Senior coding rubric

| Signal | 0 | 1 | 2 |
|---|---|---|---|
| Correctness | Không chạy | Chạy happy path | Xử lý edge cases |
| Complexity | Không biết | Biết Big-O sau khi hỏi | Chủ động chọn structure phù hợp |
| API design | Ad-hoc | Có API cơ bản | Rõ semantics + failure/cancel |
| Browser awareness | Không đề cập | Biết một API | Hiểu main thread, rendering, network |
| Testing | Không test | Happy path | Edge cases + failure modes |
| Communication | Code ngay | Giải thích khi được hỏi | Chủ động nêu assumption/trade-off |

**Senior không nhất thiết phải code nhanh nhất.** Một ứng viên mạnh làm rõ phạm vi, chọn solution đơn giản trước và biết nói rõ giới hạn của implementation.
