# Senior Frontend Interview — 199 Câu Hỏi & Trả Lời

> Không chỉ ôn "React là gì" mà đào sâu **cơ chế → reasoning → trade-off → production judgement**. Repo tổng hợp 199 câu hỏi Senior Frontend (React/Next.js/TypeScript), system design, debugging, behavioral và các bài luyện phỏng vấn thực chiến.

## 🎯 Đối tượng & cách dùng

Dành cho Middle → Senior, Senior chuẩn bị phỏng vấn công ty lớn, hoặc frontend engineer muốn hệ thống hóa kiến thức.

| Mode | Cách dùng |
|---|---|
| **Checklist** | Ôn nhanh các chương, chỉ đọc phần chưa vững |
| **Mock Interview** | Trả lời thành tiếng 2–3 phút, tự đặt follow-up, self-score |
| **Deep Dive** | Đọc cơ chế, code lại ví dụ, vẽ system design |
| **Scenario** | Giải quyết production incident trước khi xem strong answer |
| **Coding** | Code bài trong `22` rồi mới xem interviewer signals |

> **Workflow khuyến nghị:** Knowledge → Explain → Apply → Defend → Measure.

---

## 🎤 6 vòng phỏng vấn

| Vòng | Trọng tâm | Docs |
|---|---|---|
| 1. JS / TypeScript | Closure, Event Loop, Promise, generics | `01` |
| 2. React | Hooks, Fiber, concurrent, performance | `02`, `03` |
| 3. Browser / Performance | Rendering, Web Vitals, cache, bundle | `05`, `06` |
| 4. Architecture / System Design | State, boundaries, API, platform, design | `04`, `12`, `19`, `20` |
| 5. Coding / Debugging | Coding + production incidents | `13`, `18`, `22` |
| 6. Behavioral / Leadership | STAR, conflict, ownership, leadership | `14`, `15`, `21`, `23` |

---

## 📖 16 chương kiến thức gốc

| # | Chương | Số lượng | Link |
|---|---|---:|---|
| 01 | JavaScript & TypeScript | 30 | [docs/01](docs/01-javascript-typescript.md) |
| 02 | React Cơ Bản | 20 | [docs/02](docs/02-react-co-ban.md) |
| 03 | React Nâng Cao | 18 | [docs/03](docs/03-react-nang-cao.md) |
| 04 | Frontend Architecture | 17 | [docs/04](docs/04-frontend-architecture.md) |
| 05 | Performance | 19 | [docs/05](docs/05-performance.md) |
| 06 | Browser & Web Platform | 17 | [docs/06](docs/06-browser-web-platform.md) |
| 07 | Security | 13 | [docs/07](docs/07-security.md) |
| 08 | API & Networking | 16 | [docs/08](docs/08-api-networking.md) |
| 09 | State Management | 10 | [docs/09](docs/09-state-management.md) |
| 10 | Testing | 12 | [docs/10](docs/10-testing.md) |
| 11 | Next.js | 14 | [docs/11](docs/11-nextjs.md) |
| 12 | System Design | 3 bài | [docs/12](docs/12-system-design.md) |
| 13 | Debugging | 10 TH | [docs/13](docs/13-debugging.md) |
| 14 | Behavioral & Leadership | 13 | [docs/14](docs/14-behavioral-leadership.md) |
| 15 | Senior Trap Questions | 12 trap | [docs/15](docs/15-senior-trap-questions.md) |
| 16 | Lộ Trình Phỏng Vấn | — | [docs/16](docs/16-lo-trinh-phong-van.md) |

**Tổng:** 199 câu hỏi + 3 system design + 10 debugging + 12 trap.

---

## 🧠 Senior Interview Training System

| Layer | Mục đích | Docs |
|---|---|---|
| **17. Interview Framework** | Know → Explain → Apply → Defend + rubric | [17](docs/17-interview-framework.md) |
| **18. Production Scenarios** | Incident, debugging, judgement | [18](docs/18-production-scenarios.md) |
| **19. System Design Interview Mode** | Clarify → Design → Deep dive → Trade-off → Rollout | [19](docs/19-system-design-interview-mode.md) |
| **20. Frontend Platform** | Monorepo, build, CI/CD, multi-tenant, release | [20](docs/20-frontend-platform-engineering.md) |
| **21. Behavioral Story Builder** | Story bank + attack questions | [21](docs/21-behavioral-story-builder.md) |
| **22. Coding Exercises** | 12 bài + Senior coding rubric | [22](docs/22-coding-exercises.md) |
| **23. Mock Interview Scripts** | Script 30/45/60 phút + scorecard | [23](docs/23-mock-interview-scripts.md) |
| **24. Answer Anti-patterns** | Nhận diện textbook/buzzword answers | [24](docs/24-senior-answer-anti-patterns.md) |

### Công thức trả lời Senior

```text
Clarify requirement
      ↓
Classify the problem
      ↓
Choose simplest viable solution
      ↓
Explain trade-offs
      ↓
Discuss failure modes
      ↓
Plan rollout / migration
      ↓
Define success metrics
```

### Senior vs Staff

**Senior:** giải quyết tốt problem trong domain, hiểu trade-off, production failure modes và vận hành.

**Staff:** mở rộng reasoning sang ownership, team boundaries, migration cost, organizational constraints, long-term maintenance và business impact.

---

## 🧪 Mock interview

Không xem reference answer trước.

```text
Question
   ↓
2–3 min verbal answer
   ↓
Why / Trade-off / When NOT to use
   ↓
Scenario
   ↓
Follow-up attack
   ↓
Self-score 0–4
```

Sau mỗi session, ghi lại **một câu trả lời yếu nhất** và luyện lại. Mục tiêu không phải đọc được nhiều câu nhất mà là giảm số câu bạn không thể defend.

Xem `23` để chạy một mock interview hoàn chỉnh.

---

## 🧠 Tư duy Senior

> **Tại sao dùng → Giá phải trả → Khi nào KHÔNG dùng → Failure mode → Cách đo kết quả**

Tránh các anti-pattern phổ biến:

- Buzzword stacking
- Tool-first optimization
- Absolute rules
- Architecture không có migration plan
- Debug bằng phỏng đoán
- "It depends" nhưng không chốt
- Không định lượng outcome

Xem [24](docs/24-senior-answer-anti-patterns.md).

---

## 🗓 Lộ trình 4 tuần

| Tuần | Trọng tâm | Bài tập |
|---|---|---|
| 1 | React Internals + JS/TS | Vẽ mental model, coding exercises 1–4 |
| 2 | Performance + Browser | Profile một trang thật, giải scenarios 1–3 |
| 3 | Architecture + System Design + Platform | Thiết kế 3 system designs + platform scenario |
| 4 | Debugging + Behavioral + Mock | 2 mock sessions/ngày + 5 STAR stories |

Mỗi ngày ưu tiên: **nói thành tiếng → scenario/coding → feedback → đọc reference**.

---

## 🗺 Roadmap

- [x] 01–16: Core knowledge + interview roadmap
- [x] 17: Senior interview framework + rubric
- [x] 18: Production scenarios
- [x] 19: System design interview mode
- [x] 20: Frontend platform engineering
- [x] 21: Behavioral story builder
- [x] 22: Senior coding exercises
- [x] 23: Mock interview scripts
- [x] 24: Senior answer anti-patterns
- [ ] Flashcard / Anki export
- [ ] Question bank theo Senior / Staff
- [ ] Automated self-assessment / progress tracker

---

## 🤝 Đóng góp

```bash
git clone https://github.com/tungxuan1656/fe-interview.git
git checkout -b feat/your-change
# sửa docs
git commit -m "docs: improve interview training"
git push && tạo PR
```

Xem [CONTRIBUTING.md](CONTRIBUTING.md) để biết format và PR flow.

## 📄 License

MIT — dùng thoải mái cho học tập, phỏng vấn, giảng dạy.
