# Senior Frontend Interview — 199 Câu Hỏi & Trả Lời

<p align="center">
  <img src="https://img.shields.io/badge/questions-199-blue?style=for-the-badge" alt="199 questions" />
  <img src="https://img.shields.io/badge/react-18-61DAFB?style=for-the-badge&logo=react" alt="react" />
  <img src="https://img.shields.io/badge/next.js-15-black?style=for-the-badge&logo=next.js" alt="nextjs" />
  <img src="https://img.shields.io/badge/typescript-5.x-3178C6?style=for-the-badge&logo=typescript" alt="ts" />
</p>

> **Không chỉ ôn "React là gì" mà đào sâu kiến trúc, trade-off, performance, debugging, testing, security và ra quyết định như một Senior thực thụ.** Repo này tổng hợp 199 câu hỏi phỏng vấn Senior Frontend (React/Next.js/TypeScript) — mỗi câu trả lời ở mức **cơ chế → trade-off → ví dụ thực tế**, kèm system design, debugging và behavioral.

---

## 📚 Mục lục

- [Đối tượng & Cách dùng](#-đối-tượng--cách-dùng)
- [6 Vòng phỏng vấn](#-6-vòng-phỏng-vấn)
- [Tổng quan 16 chương](#-tổng-quan-16-chương)
- [Senior Interview Training System](#-senior-interview-training-system)
- [Tư duy Senior](#-tư-duy-senior)
- [Lộ trình ôn 4 tuần](#-lộ-trình-ôn-4-tuần)
- [Đóng góp](#-đóng-góp)
- [Roadmap](#-roadmap)
- [License](#-license)

---

## 🎯 Đối tượng & Cách dùng

**Dành cho:** Middle muốn lên Senior, Senior chuẩn bị phỏng vấn công ty lớn, hoặc ai muốn hệ thống lại kiến thức Frontend một cách có chiều sâu.

**3 cách dùng:**

| Cách | Khi nào | Làm sao |
|---|---|---|
| **1. Checklist** | Ôn nhanh trước phỏng vấn 1-2 ngày | Mở bảng 16 chương, tick từng câu, chỉ đọc câu chưa vững |
| **2. Mock Interview** | Luyện nói | Bốc câu hỏi, trả lời bằng miệng 2–3 phút, sau đó tự đặt follow-up |
| **3. Đào sâu** | Học để giỏi thật | Đọc từng chương, code lại ví dụ, vẽ lại system design và luyện scenario |

> 💡 Đừng chỉ đọc từ đầu đến cuối. Hãy chuyển từ **Knowledge → Engineering → Scenario → Production judgement**.

---

## 🎤 6 Vòng phỏng vấn

| Vòng | Tên | Độ quan trọng | Trọng tâm | Docs liên quan |
|---|---|---|---|---|
| 1 | **JS / TypeScript** | ⭐⭐⭐⭐ | Closure, Event Loop, TS generics | `01` |
| 2 | **React** | ⭐⭐⭐⭐⭐ | Fiber, hooks, concurrent, perf | `02`, `03` |
| 3 | **Browser / Performance** | ⭐⭐⭐⭐⭐ | Core Web Vitals, cache, bundle, virtualization | `05`, `06` |
| 4 | **Architecture / System Design** | ⭐⭐⭐⭐⭐ | FSD, state, API, micro-frontend, design | `04`, `12`, `19` |
| 5 | **Coding / Debugging** | ⭐⭐⭐⭐ | Live coding + production incidents | `13`, `18` |
| 6 | **Behavioral / Leadership** | ⭐⭐⭐⭐ | STAR, conflict, incident, mentor | `14`, `15`, `21` |

> **Thứ tự ưu tiên ôn:** React → Browser/Perf → Architecture → JS/TS → Coding/Debugging → Behavioral.

---

## 📖 Tổng quan 16 chương

| STT | Chương | Số câu | Link | Trọng tâm |
|---|---|---:|---|---|
| 01 | JavaScript & TypeScript | 30 | [docs/01-javascript-typescript.md](docs/01-javascript-typescript.md) | Closure, Event Loop, Promise, TS generics |
| 02 | React Cơ Bản | 20 | [docs/02-react-co-ban.md](docs/02-react-co-ban.md) | JSX, hooks, lifecycle, state |
| 03 | React Nâng Cao | 18 | [docs/03-react-nang-cao.md](docs/03-react-nang-cao.md) | Fiber, concurrent, memo, RSC |
| 04 | Frontend Architecture | 17 | [docs/04-frontend-architecture.md](docs/04-frontend-architecture.md) | FSD, design system, micro-frontend |
| 05 | Performance | 19 | [docs/05-performance.md](docs/05-performance.md) | Web Vitals, bundle, image, cache |
| 06 | Browser & Web Platform | 17 | [docs/06-browser-web-platform.md](docs/06-browser-web-platform.md) | Rendering, event, storage, PWA |
| 07 | Security | 13 | [docs/07-security.md](docs/07-security.md) | XSS, CSRF, CSP, auth |
| 08 | API & Networking | 16 | [docs/08-api-networking.md](docs/08-api-networking.md) | REST, GraphQL, WS, retry |
| 09 | State Management | 10 | [docs/09-state-management.md](docs/09-state-management.md) | Redux, Zustand, Query, atomic |
| 10 | Testing | 12 | [docs/10-testing.md](docs/10-testing.md) | Unit, integration, E2E, MSW |
| 11 | Next.js | 14 | [docs/11-nextjs.md](docs/11-nextjs.md) | App Router, SSR/ISR, RSC |
| 12 | System Design | 3 bài | [docs/12-system-design.md](docs/12-system-design.md) | E-commerce, Chat, Dashboard |
| 13 | Debugging | 10 TH | [docs/13-debugging.md](docs/13-debugging.md) | White screen, leak, race, iOS |
| 14 | Behavioral & Leadership | 13 | [docs/14-behavioral-leadership.md](docs/14-behavioral-leadership.md) | STAR, conflict, incident |
| 15 | Senior Trap Questions | 12 trap | [docs/15-senior-trap-questions.md](docs/15-senior-trap-questions.md) | Why React/TS/Next/Redux |
| 16 | Lộ Trình Phỏng Vấn | — | [docs/16-lo-trinh-phong-van.md](docs/16-lo-trinh-phong-van.md) | 6 vòng + 4 tuần + checklist |

**Tổng hiện tại: 199 câu hỏi + 3 system design + 10 debugging + 12 trap.**

---

## 🧠 Senior Interview Training System

Knowledge base được mở rộng bằng 5 lớp luyện phỏng vấn:

| Layer | Mục đích | Docs |
|---|---|---|
| **Interview Framework** | Know → Explain → Apply → Defend + rubric | [17](docs/17-interview-framework.md) |
| **Production Scenarios** | Luyện incident, debugging và judgement | [18](docs/18-production-scenarios.md) |
| **System Design Interview Mode** | Clarify → Design → Deep dive → Trade-off → Rollout | [19](docs/19-system-design-interview-mode.md) |
| **Frontend Platform** | Monorepo, build, CI/CD, multi-tenant, release | [20](docs/20-frontend-platform-engineering.md) |
| **Behavioral Story Builder** | Story bank + follow-up attack questions | [21](docs/21-behavioral-story-builder.md) |

### Cách luyện một câu

```text
Question
   ↓
2–3 min verbal answer
   ↓
Why / Trade-off / When NOT to use
   ↓
Production scenario
   ↓
Follow-up attack
   ↓
Self-score: 0–4
```

### Cách luyện System Design

**Không xem reference answer trước.** Hãy tự clarify requirements, đưa ra architecture, giải thích trade-off, failure modes và rollout. Sau đó mới đối chiếu `docs/12-system-design.md`.

### Cách luyện Behavioral

Không học thuộc STAR mẫu. Xây story từ trải nghiệm thật, sau đó luyện các câu hỏi đào sâu về decision, downside, outcome và lesson learned.

---

## 🧠 Tư duy Senior

Công thức cơ bản:

> **Tại sao dùng → Giá phải trả → Khi nào KHÔNG dùng → Failure mode → Cách đo kết quả**

Một câu trả lời Senior không cần chọn công nghệ phức tạp nhất. Nó cần cho thấy ứng viên biết **làm rõ requirement, chọn giải pháp vừa đủ và bảo vệ quyết định bằng trade-off**.

---

## 🗓 Lộ trình ôn 4 tuần

| Tuần | Trọng tâm | Docs | Bài tập |
|---|---|---|---|
| **1** | React Internals + JS/TS | 01, 02, 03 | Vẽ Fiber, implement debounce/deepClone |
| **2** | Performance + Browser | 05, 06, 11 | Lighthouse 1 trang thật, fix bottleneck |
| **3** | Architecture + System Design | 04, 09, 12, 19 | Thiết kế 3 bài không nhìn reference |
| **4** | Debugging + Scenarios + Behavioral | 13, 14, 18, 21 | Mock interview + production scenarios |

> Mỗi ngày: đọc → nói thành tiếng → scenario → self-score. Không chỉ đọc tài liệu thụ động.

---

## 🤝 Đóng góp

Mọi đóng góp đều được chào đón! Xem [CONTRIBUTING.md](CONTRIBUTING.md) để biết format và PR flow.

```bash
git clone https://github.com/tungxuan1656/fe-interview.git
git checkout -b feat/your-change
# sửa docs
git commit -m "docs: improve interview training"
git push && tạo PR
```

---

## 🗺 Roadmap

- [x] 01-16: Core knowledge + interview roadmap
- [x] 17: Senior interview framework + rubric
- [x] 18: Production scenarios
- [x] 19: System design interview mode
- [x] 20: Frontend platform engineering
- [x] 21: Behavioral story builder
- [ ] Thêm bài tập coding có lời giải (`exercises/`)
- [ ] Thêm flashcard Anki cho 199 câu
- [ ] Thêm mock interview scripts
- [ ] Thêm question bank theo Senior / Staff

---

## 📄 License

MIT — dùng thoải mái cho học tập, phỏng vấn, giảng dạy.
