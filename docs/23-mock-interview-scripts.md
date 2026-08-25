# 23. Mock Interview Scripts

> Dùng file này như interviewer thật: **không mở reference answer trước**, hỏi theo timeline và chỉ đưa follow-up sau khi ứng viên trả lời.

## Script A — Senior React / Performance (45 phút)

### 0–5' — Warm-up

> Hãy kể một lần bạn phải xử lý một vấn đề performance frontend thật.

Follow-up:
- Metric ban đầu là gì?
- Bạn biết bottleneck bằng bằng chứng nào?
- Thay đổi nào có impact lớn nhất?

### 5–15' — React reasoning

> Một component render chậm. Bạn bắt đầu điều tra từ đâu?

Follow-up:
- React Profiler nói gì?
- Khi nào `memo` giúp, khi nào không?
- `useMemo` có thể làm performance tệ hơn không?
- `startTransition` giải quyết vấn đề nào và không giải quyết vấn đề nào?

### 15–25' — Production scenario

> Dashboard nhận hàng nghìn events/phút. Input filter bị lag.

Không gợi ý giải pháp. Hỏi:
- Bạn cần thêm thông tin gì?
- Bottleneck có thể nằm ở đâu?
- Bạn sẽ đo gì?
- Thiết kế lại data flow thế nào?

### 25–35' — Architecture

> Team có một shared store chứa gần như mọi state. Bạn có thay library không?

Follow-up về state ownership, server state, URL state, migration và team boundaries.

### 35–42' — Coding

Chọn một bài trong `22-coding-exercises.md`. Không chấm chỉ theo code chạy; chấm communication, edge cases và API semantics.

### 42–45' — Senior signal

> Nếu được thêm một ngày để cải thiện solution, bạn sẽ thay đổi gì?

**Strong signal:** ứng viên chủ động nói về measurement, failure modes, rollout và simplification.

---

## Script B — Frontend Architecture / System Design (60 phút)

### Prompt

> Thiết kế một frontend platform phục vụ nhiều sản phẩm/tenant, có shared UI, feature flags, authentication, analytics và CI/CD.

### Interview flow

1. **Clarify (5–10')** — users, teams, deployment, tenant isolation, runtime vs build-time config.
2. **High-level design (10')** — boundaries, packages, apps, API/BFF nếu cần.
3. **State & data (10')** — server/client/URL/local state.
4. **Build & release (10')** — monorepo, cache, artifact, promotion, rollback.
5. **Runtime (10')** — observability, errors, performance, security.
6. **Trade-offs (10')** — where not to use microfrontend, where duplication is acceptable.

### Staff follow-up

> Nếu 5 team muốn deploy độc lập nhưng product muốn một trải nghiệm thống nhất, bạn ưu tiên điều gì?

> Nếu migration kéo dài 12 tháng, bạn đo migration success thế nào?

> Khi nào bạn chấp nhận duplicate code để giảm coupling?

---

## Script C — Incident / Debugging (30 phút)

### Prompt

> Sau deployment, 8% users nhận white screen. Không có crash ở backend.

### Không đưa hint ngay

Yêu cầu ứng viên:

1. Xác định blast radius.
2. Correlate release/version.
3. Reproduce.
4. Kiểm tra error monitoring, Console, Network.
5. Phân biệt JS exception, chunk load, cache và API failure.
6. Chọn rollback/mitigation.
7. Đưa ra prevention.

### Attack questions

- Nếu chỉ Safari bị?
- Nếu chỉ user từ một tenant bị?
- Nếu rollback không giải quyết?
- Nếu chunk cũ bị cache nhưng HTML mới?
- Bạn chứng minh fix an toàn thế nào?

---

## Script D — Behavioral / Leadership (30 phút)

### Core questions

1. Kể một lần bạn bất đồng với technical decision của team.
2. Kể một incident production bạn chịu trách nhiệm.
3. Kể một lần bạn mentor người khác.
4. Kể một lần bạn phải cân bằng deadline và quality.
5. Kể một quyết định bạn từng làm sai.

### Attack pattern

Sau mỗi story, hỏi liên tiếp:

```text
Why?
→ What alternatives did you consider?
→ What did you personally own?
→ What went wrong?
→ What metric changed?
→ What would you do differently?
→ Did anyone disagree with you?
```

**Không chấm câu chuyện vì nghe hay.** Chấm ownership, decision quality, self-awareness, outcome và learning.

---

## Full mock scorecard

| Dimension | 1 | 2 | 3 | 4 |
|---|---|---|---|---|
| Technical depth | Memorized | Basic | Deep | Mechanism + boundaries |
| Reasoning | Random | Some logic | Structured | Requirement-driven |
| Trade-off | None | One-sided | Balanced | Explicit cost + reversibility |
| Production | Textbook | Generic example | Real debugging | Metrics + rollout + incident thinking |
| Communication | Rambling | Understandable | Clear | Concise, adaptive, interviewer-aware |
| Ownership | Task-focused | Feature-focused | System-focused | Business/team/system impact |

### Suggested interpretation

- **0–12:** chưa đạt Senior signal.
- **13–18:** Middle+ / Senior chưa ổn định.
- **19–21:** Senior khá rõ.
- **22–24:** Senior mạnh; có một số Staff signals.
