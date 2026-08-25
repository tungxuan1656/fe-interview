# 16. Lộ Trình Phỏng Vấn - 6 Vòng & Kế Hoạch 4 Tuần

> Nếu ôn theo **phỏng vấn thật**, đừng ôn dàn trải 199 câu (~211 thực tế gồm 12 trap, xem README). Hãy ôn theo **6 vòng** mà công ty Senior hay hỏi, với độ quan trọng ⭐ khác nhau. Chương này cho bạn **bản đồ 6 vòng + lộ trình 4 tuần cho background React/React Native + checklist tự đánh giá**.

## Mục lục

- [6 Vòng Phỏng Vấn Senior Frontend](#6-vòng-phỏng-vấn-senior-frontend)
- [Vòng 1: JavaScript / TypeScript ⭐⭐⭐⭐](#vòng-1-javascript--typescript-)
- [Vòng 2: React ⭐⭐⭐⭐⭐](#vòng-2-react-)
- [Vòng 3: Browser / Performance ⭐⭐⭐⭐⭐](#vòng-3-browser--performance-)
- [Vòng 4: Architecture / System Design ⭐⭐⭐⭐⭐](#vòng-4-architecture--system-design-)
- [Vòng 5: Coding / Debugging ⭐⭐⭐⭐](#vòng-5-coding--debugging-)
- [Vòng 6: Behavioral / Leadership ⭐⭐⭐⭐](#vòng-6-behavioral--leadership-)
- [Lộ trình 4 tuần cho background React / React Native](#lộ-trình-4-tuần-cho-background-react--react-native)
- [Checklist tự đánh giá](#checklist-tự-đánh-giá)

---

## 6 Vòng Phỏng Vấn Senior Frontend

| Vòng | Tên | Độ quan trọng | Thời lượng | Hình thức |
|---|---|---|---|---|
| 1 | JS / TS | ⭐⭐⭐⭐ | 45-60' | Hỏi đáp + code nhỏ |
| 2 | React | ⭐⭐⭐⭐⭐ | 60' | Hỏi sâu + code |
| 3 | Browser / Performance | ⭐⭐⭐⭐⭐ | 45-60' | Hỏi đáp + case |
| 4 | Architecture / System Design | ⭐⭐⭐⭐⭐ | 60' | Whiteboard |
| 5 | Coding / Debugging | ⭐⭐⭐⭐ | 60-90' | Live coding |
| 6 | Behavioral / Leadership | ⭐⭐⭐⭐ | 30-45' | STAR story |

> **Thứ tự ưu tiên ôn:** React (⭐⭐⭐⭐⭐) → Browser/Perf (⭐⭐⭐⭐⭐) → Architecture (⭐⭐⭐⭐⭐) → JS/TS (⭐⭐⭐⭐) → Coding/Debugging (⭐⭐⭐⭐) → Behavioral (⭐⭐⭐⭐). Đừng ôn JS 2 tuần mà bỏ React internals.

---

### Vòng 1: JavaScript / TypeScript ⭐⭐⭐⭐

**Mục tiêu:** Kiểm tra **nền tảng** — có hiểu cơ chế hay chỉ dùng.

**Câu hỏi trọng tâm (từ docs/01):**

- Closure, Event Loop (micro/macro), `this`, Prototype, Hoisting/TDZ
- `Promise` vs `async/await`, `call/apply/bind`, `debounce/throttle`
- TS: `any` vs `unknown`, `type` vs `interface`, Generics, Conditional Types, Utility Types
- Monorepo perf TS (project references, `isolatedModules`)

**Cách chấm điểm:**

| Level | Biểu hiện |
|---|---|
| Junior | Trả lời định nghĩa, không nói được ví dụ |
| Middle | Nói được ví dụ, chưa nói trade-off |
| **Senior** | **Giải thích cơ chế (engine), trade-off, ví dụ production, khi nào không dùng** |

**Tips:**

- Đừng học thuộc định nghĩa — hãy **vẽ** Event Loop, Prototype chain
- Mỗi câu chuẩn bị 1 **code snippet** demo
- TS: chuẩn bị **tự implement `Pick`, `Omit`, `ReturnType`**

**Docs:** `docs/01-javascript-typescript.md` (Câu 1-30)

---

### Vòng 2: React ⭐⭐⭐⭐⭐

**Mục tiêu:** Vòng quan trọng nhất — phân biệt Senior thật/giả.

**Câu hỏi trọng tâm (từ docs/02, 03):**

- **Internals:** Reconciliation, Fiber, Batching, Concurrent (Suspense, Transition, `useDeferredValue`)
- **Hooks:** `useEffect` vs `useLayoutEffect`, `useMemo`/`useCallback` khi nào cần, `useRef` vs `useState`, custom hook
- **Performance:** Vì sao re-render, `memo` + `useMemo` + `useCallback` đúng chỗ, Context hell
- **Pattern:** Compound component, Render props, HOC vs Hook, Error Boundary
- **RSC:** Server vs Client Component, khi nào dùng gì

**Cách chấm điểm:**

- **Fail:** Không giải thích được vì sao `setState` async, không biết `key` để làm gì
- **Pass Senior:** Nói được **Fiber**, **batching 18**, **concurrent**, **trade-off của memo**, và **khi nào KHÔNG memo**

**Tips (cho background React Native):**

- RN cũng dùng Fiber, bridge → hiểu React internals giúp debug RN tốt hơn
- Chuẩn bị câu **"React Native khác React web thế nào?"** (bridge, JSI, Fabric)
- Nhấn mạnh **performance RN** (FlatList virtualization, `useMemo` cho expensive calc)

**Docs:** `docs/02-react-co-ban.md`, `docs/03-react-nang-cao.md`

---

### Vòng 3: Browser / Performance ⭐⭐⭐⭐⭐

**Mục tiêu:** Kiểm tra **hiểu platform**, không chỉ React.

**Câu hỏi trọng tâm (từ docs/05, 06):**

- **Core Web Vitals:** LCP, INP, CLS — ngưỡng, cách đo, cách fix
- **Rendering:** Critical Rendering Path, Reflow/Repaint, Compositor, `transform` vs `top`
- **Cache:** HTTP Cache (`Cache-Control`, `ETag`), CDN, SW Cache
- **Bundle:** Code splitting, tree-shaking, `import()` dynamic, `source-map-explorer`
- **Image/Font:** `srcset`, `preload`, `font-display`, `next/image`
- **Virtualization:** Khi nào cần, tanstack-virtual vs react-window

**Cách chấm điểm:**

- Senior phải nói được **số liệu** (LCP <2.5s, bundle <200kb), **tool** (Lighthouse, Performance tab, Coverage), và **trade-off** (preload 1 LCP thôi)

**Tips:**

- Chuẩn bị 1 case **"Page load 8s bạn fix sao?"** (đo → bundle → image → code split → cache)
- Nhớ **4 nhóm bottleneck**: Network, Compute, Render, Resource

**Docs:** `docs/05-performance.md`, `docs/06-browser-web-platform.md`

---

### Vòng 4: Architecture / System Design ⭐⭐⭐⭐⭐

**Mục tiêu:** Vòng **quyết định offer Senior** — không code, chỉ thiết kế.

**Câu hỏi trọng tâm (từ docs/04, 12):**

- **Project structure:** Feature-based vs Layer-based, FSD, tránh cross-feature import
- **State:** Server vs Client state, khi nào Zustand/Redux/Query
- **API Layer:** BFF, retry, interceptor, contract (OpenAPI + zod)
- **System Design:** E-commerce, Chat realtime, Dashboard big data (3 bài trong docs/12)
- **Micro-frontend / Monorepo:** Khi nào dùng, trade-off
- **Refactor:** Strangler fig, feature flag, canary

**Cách chấm điểm:**

| Level | Biểu hiện |
|---|---|
| Junior | Vẽ UI, không nói state/API/cache |
| Middle | Nói được component, chưa nói trade-off |
| **Senior** | **Vẽ state + API + cache + pagination + failure mode, nói được trade-off của từng quyết định** |

**Tips:**

- Luyện **3 bài docs/12** bằng cách **vẽ lại** không nhìn doc, rồi tự hỏi follow-up
- Chuẩn bị **ADR template** để trả lời "vì sao chọn X?"
- Nói **"Tùy scale"** — app 10 trang khác 100k SKU

**Docs:** `docs/04-frontend-architecture.md`, `docs/12-system-design.md`

---

### Vòng 5: Coding / Debugging ⭐⭐⭐⭐

**Mục tiêu:** Kiểm tra **tay nghề** — code sạch + debug có hệ thống.

**Câu hỏi trọng tâm (từ docs/13):**

- **Coding:** Debounce, Deep clone, Promise pool, Virtual list, Autocomplete (thường cho 45-60')
- **Debugging:** White screen, memory leak, race condition (search sai thứ tự), double submit, iOS crash

**Cách chấm điểm:**

- Code: **chạy được + edge case + complexity + test**
- Debug: **quy trình 11 bước**, không đoán mò

**Tips:**

- Luyện **LeetCode easy/medium** (2-3 bài/tuần) + **hand-write** (không autocomplete)
- Debugging: nhớ **Reproduce → Logs → Monitoring → ... → Monitor**, mỗi bug chuẩn bị **checklist**
- Chuẩn bị câu **"Kể 1 bug khó nhất bạn từng fix"** (STAR)

**Docs:** `docs/13-debugging.md`

---

### Vòng 6: Behavioral / Leadership ⭐⭐⭐⭐

**Mục tiêu:** Kiểm tra **con người** — có làm được Senior trong team không.

**Câu hỏi trọng tâm (từ docs/14, 15):**

- Review code, conflict, technical debt, deadline, quyết định sai, thuyết phục, PM/BE conflict, incident, mentor, delegate
- Trap: Why React/TS/Next.js/Redux... — trả lời bằng **trade-off**

**Cách chấm điểm:**

- Dùng **STAR** + **số liệu** + **lesson learned**. Không kể chung chung "mình là người teamwork tốt"

**Tips:**

- Chuẩn bị **5 story STAR** (conflict, fail, lead, incident, mentor) dùng lại cho nhiều câu
- Trap: luôn trả lời **"Tại sao + Trade-off + Khi nào KHÔNG dùng"**

**Docs:** `docs/14-behavioral-leadership.md`, `docs/15-senior-trap-questions.md`

---

## Lộ trình 4 tuần cho background React / React Native

> Nhấn mạnh: **React internals → Performance → Architecture → System Design → Debugging → Leadership**. Đừng ôn lan man.

| Tuần | Trọng tâm | Docs | Mục tiêu | Bài tập |
|---|---|---|---|---|
| **1** | **React Internals + JS/TS nền** | 01, 02, 03 | Nắm Fiber, hooks, TS | Đọc 02+03, tự vẽ Fiber, implement `useDebounce`, `deepClone` |
| **2** | **Performance + Browser** | 05, 06, 11 | LCP/INP/CLS, cache, Next.js | Lighthouse 1 trang thật, fix LCP, học `next/image`, `loading` |
| **3** | **Architecture + System Design** | 04, 09, 12 | Thiết kế state/API/cache | Vẽ lại 3 bài system design không nhìn doc, viết 1 ADR |
| **4** | **Debugging + Behavioral + Mock** | 13, 14, 15, 07, 08, 10 | Debug hệ thống + STAR + trap | Mock interview 2 vòng/ngày, chuẩn bị 5 STAR story, ôn trap table |

**Lịch chi tiết:**

- **Mỗi ngày:** 2h đọc docs + 1h code/debug + 30' kể STAR bằng miệng (record lại)
- **Cuối tuần:** 1 buổi mock interview full 6 vòng (nhờ bạn hoặc tự record)
- **Tuần 4:** Tập trung **nói** hơn **đọc** — phỏng vấn là nói, không phải viết

**Cho background React Native:**

- Tuần 1: Thêm câu "RN bridge/JSI/Fabric khác web thế nào?" + FlatList perf
- Tuần 2: So sánh **Web Vitals** vs **RN perf** (FPS, bridge, Hermes)
- Tuần 3: System design cho **mobile app** (offline-first, sync, push)
- Tuần 4: Chuẩn bị câu **"Vì sao chuyển từ RN sang web / làm cả 2?"**

---

## Checklist tự đánh giá

Tự chấm 0-3 cho mỗi mục (0=chưa biết, 3=dạy được người khác). **Pass Senior khi ≥2 ở mọi mục ⭐⭐⭐⭐⭐ và ≥1.5 ở còn lại.**

### JS/TS

- [ ] Giải thích Event Loop + microtask bằng code
- [ ] Implement `debounce`, `throttle`, `deepClone` không nhìn
- [ ] Phân biệt `any`/`unknown`/`never`, tự viết `Pick`/`Omit`

### React

- [ ] Vẽ Fiber/reconciliation, giải thích batching 18
- [ ] Nói khi nào `useMemo` cần/không cần (có số liệu)
- [ ] Phân biệt Server vs Client Component (RSC)

### Browser/Perf

- [ ] Nêu ngưỡng LCP/INP/CLS và cách fix từng cái
- [ ] Dùng Performance tab tìm long task + layout thrashing
- [ ] Giải thích `Cache-Control`, `ETag`, `stale-while-revalidate`

### Architecture

- [ ] Vẽ kiến trúc E-commerce (state, API, cache, pagination) không nhìn doc
- [ ] Trả lời "khi nào dùng Zustand vs Redux vs Query"
- [ ] Viết ADR cho 1 quyết định

### Coding/Debugging

- [ ] Code 1 bài medium trong 30' (chạy + test)
- [ ] Kể quy trình debug white screen 11 bước
- [ ] Fix được race condition search + double submit

### Behavioral

- [ ] Kể 5 story STAR trôi chảy <2' mỗi story
- [ ] Trả lời trap "Why React?" bằng trade-off

**Mức độ sẵn sàng:**

| Điểm TB | Đánh giá | Hành động |
|---|---|---|
| <1.0 | Chưa sẵn sàng | Ôn thêm 2 tuần, tập trung mục thấp |
| 1.0-2.0 | Middle sẵn sàng | Mock thêm, luyện nói |
| **≥2.0** | **Senior sẵn sàng** | **Book phỏng vấn!** |

> **Lời khuyên cuối:** Đừng ôn để thuộc — ôn để **hiểu và kể được**. Phỏng vấn Senior là **kể chuyện có chiều sâu**, không phải thi trắc nghiệm.
