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

Yêu cầu: trailing invocation, `cancel`, `flush`, giữ `this` và arguments.

### Follow-up

Leading/trailing mode? Nếu `fn` throw? Search UI có cần AbortController không?

### Interviewer looks for

Closure, timer lifecycle, API semantics và khả năng phân biệt debounce với request cancellation.

---

## Exercise 02 — Concurrent request limiter

Giới hạn tối đa `N` async tasks chạy đồng thời.

```ts
mapWithConcurrency(tasks, 3)
```

### Follow-up

Preserve order? Reject một task có dừng toàn bộ không? Cancellation? Retry có vượt concurrency không?

### Interviewer looks for

Queue, scheduling, failure semantics và resource control.

---

## Exercise 03 — LRU cache

Implement generic `LRUCache<K, V>` với `get`, `set`, `has`, `delete`, capacity cố định và `O(1)` cho `get/set`.

### Follow-up

TTL có nên nằm cùng abstraction? Cache server state khác memoization thế nào?

### Interviewer looks for

Data structure choice và khả năng tránh over-engineering.

---

## Exercise 04 — Request deduplication

Nếu 10 component cùng request `/api/user/123` trong cùng thời điểm, chỉ gửi một request và share Promise.

### Follow-up

Reject xử lý thế nào? Request lifetime? Một consumer cancel có được abort shared request không?

### Interviewer looks for

In-flight cache, ownership và cancellation semantics.

---

## Exercise 05 — Stale response protection

Implement search request manager sao cho response cũ không overwrite kết quả mới.

### Follow-up

So sánh sequence number, AbortController, query library và server-side request IDs.

### Interviewer looks for

Hiểu rằng cancellation và stale-response protection là hai vấn đề liên quan nhưng không đồng nhất.

---

## Exercise 06 — Virtualized list

Thiết kế algorithm render 1,000,000 rows nhưng chỉ mount viewport + overscan.

### Follow-up

Variable row height? Scroll restore? Keyboard navigation? Accessibility?

### Interviewer looks for

Measurement, overscan, rendering cost và UX constraints.

---

## Exercise 07 — Event emitter typed

Implement `on`, `off`, `once`, `emit` với TypeScript typed events.

### Follow-up

Listener tự unsubscribe trong lúc emit? Listener throw? Memory leak detection?

### Interviewer looks for

Mutation trong iteration, lifecycle và type-safe API design.

---

## Exercise 08 — Deep equality

Implement `isEqual(a, b)` cho object/array/primitive.

### Follow-up

Circular reference? Date/Map/Set? NaN và `-0`? Vì sao JSON.stringify không phải general solution?

### Interviewer looks for

Scope trước code và khả năng nói rõ giới hạn implementation.

---

## Exercise 09 — Retry with backoff

Implement retry với exponential backoff và maximum attempts.

### Follow-up

Retry status nào? Jitter? POST có retry an toàn không? Timeout + retry?

### Interviewer looks for

Idempotency, retry storms, timeout và backoff.

---

## Exercise 10 — Tiny state store

Implement:

```ts
const store = createStore(initialState)
store.getState()
store.setState(...)
store.subscribe(listener)
```

### Follow-up

Selector subscription? Prevent unnecessary renders? Batched updates? Persistence? Server state có nên vào store không?

### Interviewer looks for

State ownership, subscriptions và rendering implications hơn syntax.

---

## Exercise 11 — Priority task scheduler

Thiết kế scheduler `high | normal | low` và tránh block main thread quá lâu.

### Follow-up

`requestIdleCallback` có đủ không? Yielding? Khi nào `startTransition` phù hợp hơn?

### Interviewer looks for

Browser main-thread model và distinction giữa application scheduler với React scheduler.

---

## Exercise 12 — Large response processing

API trả 100,000 records. Transform để hiển thị nhưng không freeze UI.

### Follow-up

Chunk work? Web Worker? Streaming? Memoization? Có cần transform toàn bộ data không?

### Interviewer looks for

Tối ưu pipeline thay vì chỉ tối ưu một function.

---

## Senior coding rubric

| Signal | 0 | 1 | 2 |
|---|---|---|---|
| Correctness | Không chạy | Happy path | Edge cases |
| Complexity | Không biết | Biết sau khi hỏi | Chủ động chọn structure |
| API design | Ad-hoc | API cơ bản | Rõ semantics + failure/cancel |
| Browser awareness | Không đề cập | Biết API | Hiểu main thread/render/network |
| Testing | Không test | Happy path | Edge + failure modes |
| Communication | Code ngay | Giải thích khi hỏi | Chủ động assumption/trade-off |

**Senior không nhất thiết code nhanh nhất.** Ứng viên mạnh làm rõ phạm vi, chọn solution đơn giản trước và nói rõ giới hạn implementation.
