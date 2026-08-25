# Senior Frontend Interview — 199 Câu Hỏi & Trả Lời

<p align="center">
  <img src="https://img.shields.io/badge/questions-199-blue?style=for-the-badge" alt="199 questions" />
  <img src="https://img.shields.io/badge/react-18-61DAFB?style=for-the-badge&logo=react" alt="react" />
  <img src="https://img.shields.io/badge/next.js-15-black?style=for-the-badge&logo=next.js" alt="nextjs" />
  <img src="https://img.shields.io/badge/typescript-5.x-3178C6?style=for-the-badge&logo=typescript" alt="ts" />
</p>

<p align="center">
  <img src="https://img.shields.io/github/stars/you/fe-interview-questions?style=social" alt="stars" />
  <img src="https://img.shields.io/github/forks/you/fe-interview-questions?style=social" alt="forks" />
  <img src="https://img.shields.io/badge/last%20update-Aug%202026-green" alt="last update" />
  <img src="https://img.shields.io/badge/license-MIT-yellow" alt="license" />
</p>

> **Không chỉ ôn "React là gì" mà đào sâu kiến trúc, trade-off, performance, debugging, testing, security và ra quyết định như một Senior thực thụ.** Repo này tổng hợp 199 câu hỏi phỏng vấn Senior Frontend (React/Next.js/TypeScript) — mỗi câu trả lời ở mức **cơ chế → trade-off → ví dụ thực tế**, kèm system design, debugging và behavioral.

---

## 📚 Mục lục

- [Đối tượng & Cách dùng](#-đối-tượng--cách-dùng)
- [6 Vòng phỏng vấn](#-6-vòng-phỏng-vấn)
- [Tổng quan 16 chương](#-tổng-quan-16-chương)
- [Tư duy Senior: trả lời trap](#-tư-duy-senior-trả-lời-trap)
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
| **2. Mock Interview** | Luyện nói | Mỗi ngày bốc 5 câu ngẫu nhiên, trả lời bằng miệng 2' / câu, record lại |
| **3. Đào sâu** | Học để giỏi thật | Đọc từng chương theo lộ trình 4 tuần, code lại ví dụ, vẽ lại system design |

> 💡 Tips: Đừng đọc từ đầu đến cuối như sách giáo khoa. Hãy bắt đầu từ chương bạn **yếu nhất** (thường là Performance hoặc System Design).

---

## 🎤 6 Vòng phỏng vấn

Đây là cấu trúc phỏng vấn Senior thực tế tại các công ty lớn. Mỗi vòng có độ quan trọng khác nhau:

| Vòng | Tên | Độ quan trọng | Trọng tâm | Docs liên quan |
|---|---|---|---|---|
| 1 | **JS / TypeScript** | ⭐⭐⭐⭐ | Closure, Event Loop, TS generics | `01` |
| 2 | **React** | ⭐⭐⭐⭐⭐ | Fiber, hooks, concurrent, perf | `02`, `03` |
| 3 | **Browser / Performance** | ⭐⭐⭐⭐⭐ | Core Web Vitals, cache, bundle, virtualization | `05`, `06` |
| 4 | **Architecture / System Design** | ⭐⭐⭐⭐⭐ | FSD, state, API, micro-frontend, 3 bài lớn | `04`, `12` |
| 5 | **Coding / Debugging** | ⭐⭐⭐⭐ | Live coding + 10 tình huống debug | `13` |
| 6 | **Behavioral / Leadership** | ⭐⭐⭐⭐ | STAR, conflict, incident, mentor | `14`, `15` |

> **Thứ tự ưu tiên ôn:** React (⭐⭐⭐⭐⭐) → Browser/Perf (⭐⭐⭐⭐⭐) → Architecture (⭐⭐⭐⭐⭐) → JS/TS (⭐⭐⭐⭐) → Coding/Debugging (⭐⭐⭐⭐) → Behavioral (⭐⭐⭐⭐)

Chi tiết mỗi vòng (mục tiêu, câu hỏi, cách chấm điểm): xem `docs/16-lo-trinh-phong-van.md`.

---

## 📖 Tổng quan 16 chương

| STT | Chương | Số câu | Link | Trọng tâm |
|---|---|---|---|---|
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
| 15 | Senior Trap Questions | 12 trap | [docs/15-senior-trap-questions.md](docs/15-senior-trap-questions.md) | Why React/TS/Next/Redux... |
| 16 | Lộ Trình Phỏng Vấn | — | [docs/16-lo-trinh-phong-van.md](docs/16-lo-trinh-phong-van.md) | 6 vòng + 4 tuần + checklist |

**Tổng: 199 câu hỏi + 3 bài system design + 10 tình huống debug + 12 trap**

Mỗi file `docs/*.md` có cấu trúc: **H1 + mô tả + TOC + từng câu (### Câu X) với Trả lời Senior → Trade-off → Ví dụ → Câu hỏi đào sâu**.

---

## 🧠 Tư duy Senior: trả lời trap

Điểm khác biệt lớn nhất giữa Junior và Senior không phải là biết nhiều hơn, mà là **trả lời bằng trade-off**.

| Junior | Senior |
|---|---|
| "Dùng Redux vì nó tốt" | "Dùng Redux khi state global phức tạp + team lớn cần convention, nhưng trade-off là boilerplate; với app nhỏ thì Zustand + Query gọn hơn" |
| "SSR luôn nhanh hơn" | "SSR giúp LCP nhanh nhưng TTFB chậm + hydration cost; chỉ SSR khi cần SEO + dynamic" |
| "Tailwind là best" | "Tailwind giúp velocity nhưng HTML dài; khi cần CSS phức tạp hoặc theme runtime thì CSS Modules hợp hơn" |

**Công thức Senior cho mọi câu "Why X?":**

> **Tại sao dùng → Giá phải trả (trade-off) → Khi nào KHÔNG dùng**

Luyện 12 trap trong `docs/15-senior-trap-questions.md` để không bị bắt bài.

---

## 🗓 Lộ trình ôn 4 tuần

Cho background **React / React Native**, nhấn mạnh **React internals → Performance → Architecture → System Design → Debugging → Leadership**:

| Tuần | Trọng tâm | Docs | Bài tập |
|---|---|---|---|
| **1** | React Internals + JS/TS | 01, 02, 03 | Vẽ Fiber, implement debounce/deepClone |
| **2** | Performance + Browser | 05, 06, 11 | Lighthouse 1 trang thật, fix LCP |
| **3** | Architecture + System Design | 04, 09, 12 | Vẽ lại 3 bài system design không nhìn doc |
| **4** | Debugging + Behavioral + Mock | 13, 14, 15 | Mock interview 2 vòng/ngày, 5 STAR story |

> Mỗi ngày: 2h đọc + 1h code + 30' kể STAR bằng miệng. Cuối tuần mock full 6 vòng.

Chi tiết + checklist tự đánh giá: xem `docs/16-lo-trinh-phong-van.md`.

---

## 🤝 Đóng góp

Mọi đóng góp đều được chào đón! Xem [CONTRIBUTING.md](CONTRIBUTING.md) để biết:

- Cách thêm câu hỏi/đáp án
- Format markdown chuẩn
- PR flow

```bash
# Quick start
git clone https://github.com/you/fe-interview-questions.git
git checkout -b feat/cau-200
# sửa docs/xx-xxx.md
git commit -m "feat(docs): thêm câu 200 về ..."
git push && tạo PR
```

---

## 🗺 Roadmap

- [x] 01-12: Core (JS, React, Arch, Perf, Browser, Security, API, State, Testing, Next.js, System Design)
- [x] 13: Debugging (10 tình huống)
- [x] 14: Behavioral (13 câu STAR)
- [x] 15: Senior Trap (12 trap)
- [x] 16: Lộ trình 4 tuần + checklist
- [ ] Thêm **bài tập coding** có lời giải (`exercises/`)
- [ ] Thêm **flashcard** Anki cho 199 câu
- [ ] Video mock interview

---

## 📄 License

MIT — dùng thoải mái cho học tập, phỏng vấn, giảng dạy. Nếu thấy hữu ích, hãy ⭐ **star repo** để ủng hộ!

---

<p align="center">
  <i>Cảm ơn bạn đã đọc đến đây. Chúc bạn pass Senior với offer xịn! 🚀</i><br/>
  <b>Nếu repo giúp bạn, hãy tặng 1 ⭐ — đó là động lực lớn nhất cho maintainer.</b>
</p>
