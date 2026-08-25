# 17. Senior Interview Framework

> Biến knowledge base thành một hệ thống luyện phỏng vấn: **Know → Explain → Apply → Defend**.

## 1. Mục tiêu

Một Senior Frontend không chỉ trả lời đúng định nghĩa. Câu trả lời tốt cần thể hiện:

1. **Mechanism** — hiểu hệ thống hoạt động thế nào.
2. **Reasoning** — biết vì sao chọn giải pháp.
3. **Trade-off** — biết cái giá phải trả.
4. **Application** — áp dụng được vào production.
5. **Judgement** — biết khi nào không nên dùng.

## 2. Ba tầng cho mỗi câu hỏi

### Level 1 — Knowledge

- What is it?
- How does it work?
- What problem does it solve?

### Level 2 — Engineering

- Why choose it?
- What are the trade-offs?
- What alternatives exist?
- When should you not use it?

### Level 3 — Scenario

- Production đang gặp vấn đề gì?
- Bạn sẽ đo/debug từ đâu?
- Bạn cần hỏi thêm requirement nào?
- Bạn sẽ triển khai thay đổi thế nào mà không phá hệ thống?

## 3. Rubric

| Level | Đặc điểm câu trả lời |
|---|---|
| **Junior** | Nhớ định nghĩa, API và ví dụ cơ bản. |
| **Middle** | Giải thích được cách dùng và một số trade-off. |
| **Senior** | Bắt đầu từ requirement, phân loại vấn đề, nêu trade-off, failure mode và cách vận hành production. |
| **Staff** | Nhìn cả hệ thống: ownership, team boundary, migration cost, observability, long-term maintenance và business impact. |

## 4. Công thức trả lời Senior

```text
Clarify requirement
      ↓
Classify the problem
      ↓
State the simplest viable solution
      ↓
Explain trade-offs
      ↓
Discuss failure modes
      ↓
Explain rollout / migration
      ↓
Define how success is measured
```

Không cần cố đưa ra kiến trúc phức tạp ngay từ đầu. Interviewer thường đánh giá cao việc ứng viên **làm rõ bài toán trước khi chọn công nghệ**.

## 5. Mock interview mode

Khi luyện một câu:

1. Đọc **chỉ câu hỏi**.
2. Trả lời bằng miệng trong 2–3 phút.
3. Tự đặt ít nhất 2 follow-up.
4. Đối chiếu reference answer.
5. Chấm bản thân theo 4 tiêu chí:
   - Accuracy
   - Reasoning
   - Trade-off
   - Production experience

### Điểm gợi ý

- **0–1:** Chưa vững.
- **2:** Biết khái niệm nhưng còn textbook.
- **3:** Senior-level reasoning.
- **4:** Có production judgement và trade-off rõ ràng.

## 6. Senior answer vs textbook answer

### Textbook

> Redux phù hợp khi ứng dụng có state global phức tạp.

### Senior

> Tôi không chọn state library trước. Tôi phân loại state thành server state, URL state, local UI state và shared client state. Nếu vấn đề chủ yếu là server state thì Query phù hợp hơn Redux. Nếu shared client state cần convention mạnh, middleware, devtools và predictable updates trong team lớn thì Redux Toolkit có lợi thế. Với state đơn giản và ownership rõ, Zustand có thể giảm ceremony.

## 7. Checklist trước khi kết thúc câu trả lời

- [ ] Tôi đã nói rõ assumption chưa?
- [ ] Tôi đã giải thích mechanism chưa?
- [ ] Tôi đã nói trade-off chưa?
- [ ] Tôi có nêu trường hợp **không nên dùng** chưa?
- [ ] Tôi có production example hoặc failure mode chưa?
- [ ] Tôi có nói cách đo kết quả chưa?
