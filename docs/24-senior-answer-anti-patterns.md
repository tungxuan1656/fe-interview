# 24. Senior Answer Anti-patterns

> Một số câu trả lời nghe có vẻ senior nhưng thực tế chưa thể hiện engineering judgement.

## 1. Buzzword stacking

> "Dùng Next.js + RSC + Edge + Redis + Kafka + microfrontend để scale."

**Vấn đề:** chưa chứng minh requirement cần những thứ đó.

**Senior:** bắt đầu từ constraints, chọn simplest viable architecture rồi chỉ thêm complexity khi có evidence.

---

## 2. Tool-first thinking

> "Performance kém thì thêm `useMemo`."

**Vấn đề:** optimization trước measurement.

**Senior:** reproduce → profile → identify bottleneck → change → measure.

---

## 3. Absolute rules

Các câu như:

- "SSR luôn tốt hơn CSR."
- "Redux không nên dùng nữa."
- "Microfrontend là dành cho app lớn."
- "Memoize mọi component."
- "Monorepo luôn tốt hơn polyrepo."

đều thiếu context.

**Senior:** nói rõ condition, cost và boundary của quyết định.

---

## 4. False precision

> "App trên 50 components thì dùng FSD."

Component count không phải một architecture threshold đáng tin cậy.

Nên nhìn:

- domain complexity
- coupling
- ownership
- change frequency
- deployment boundaries
- coordination cost

---

## 5. Architecture without migration

Thiết kế target architecture rất đẹp nhưng không trả lời:

> "Production hiện tại đang chạy. Bạn chuyển từ A sang B thế nào?"

Senior cần nói về:

- incremental migration
- compatibility layer
- feature flags
- strangler pattern khi phù hợp
- observability
- rollback
- success metrics

---

## 6. Debug bằng phỏng đoán

> "Chắc WebSocket làm lag."

Senior không kết luận từ symptom. Họ phân biệt:

```text
Network ingestion
→ state updates
→ scheduling
→ React render
→ DOM/layout/paint
→ input responsiveness
```

---

## 7. Chỉ nói happy path

Một design tốt phải trả lời tối thiểu:

- What happens when API fails?
- What happens when data is stale?
- What happens when deployment is partially rolled out?
- What happens when user retries?
- What happens when one dependency is slow?

---

## 8. Confusing cancellation with correctness

`AbortController` có thể giảm work/request không cần thiết nhưng không tự động giải quyết mọi stale-data problem.

Senior cần phân biệt:

- cancellation
- deduplication
- ordering
- stale response protection
- cache invalidation

---

## 9. "It depends" nhưng không chốt

> "Redux hay Zustand? It depends."

Nếu dừng ở đó thì chưa đủ.

Cần nói:

```text
Given X constraints → I choose A
because Y
Cost is Z
If constraint changes to W → I would choose B
```

---

## 10. Experience dumping

> "Tôi đã làm 5 năm React nên..."

Thâm niên không phải evidence của judgement.

Thay bằng:

> Situation → decision → alternatives → outcome → lesson.

---

## 11. Overclaiming

Nếu không biết, nói rõ:

> "Phần implementation detail này tôi không nhớ chính xác; mental model của tôi là... Tôi sẽ verify bằng source/docs trước khi thay đổi production."

Đây thường tốt hơn việc đoán một chi tiết kỹ thuật.

---

## 12. Không định lượng outcome

Thay vì:

> "Performance tốt hơn nhiều."

Nói:

> "Sau rollout, metric X giảm từ A xuống B ở nhóm users Y."

Nếu không có metric, nói rõ đó là qualitative outcome thay vì bịa số.

---

## Quick self-check

Trước khi kết thúc câu trả lời, tự hỏi:

- [ ] Tôi có assumption rõ chưa?
- [ ] Tôi có evidence trước khi kết luận chưa?
- [ ] Tôi có alternative không?
- [ ] Tôi có trade-off không?
- [ ] Tôi có failure mode không?
- [ ] Tôi có migration/rollback không?
- [ ] Tôi có metric outcome không?
- [ ] Tôi có nói khi nào giải pháp của mình không phù hợp không?
