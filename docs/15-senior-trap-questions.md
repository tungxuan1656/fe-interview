# 15. Senior Trap Questions - Những Câu Bẫy Phân Biệt Junior vs Senior

> Nhóm câu "trap" (bẫy) không hỏi kiến thức mà hỏi **tư duy ra quyết định**. Junior trả lời "vì nó tốt/hot", Senior trả lời "vì **trade-off** và **khi nào KHÔNG dùng**". Mỗi câu gồm: Câu hỏi bẫy → Câu trả lời Junior sai → **Trả lời Senior đúng** (tại sao + trade-off + khi nào không dùng).

## Mục lục

- [Trap 1: Why React?](#trap-1-why-react)
- [Trap 2: Why TypeScript?](#trap-2-why-typescript)
- [Trap 3: Why Next.js?](#trap-3-why-nextjs)
- [Trap 4: Why Redux / Zustand / Jotai?](#trap-4-why-redux--zustand--jotai)
- [Trap 5: Why Micro-frontend?](#trap-5-why-micro-frontend)
- [Trap 6: Why Monorepo?](#trap-6-why-monorepo)
- [Trap 7: Why SSR / SSG / ISR?](#trap-7-why-ssr--ssg--isr)
- [Trap 8: Why Tailwind?](#trap-8-why-tailwind)
- [Trap 9: Why NOT Tailwind? (Khi nào không dùng)](#trap-9-why-not-tailwind-khi-nào-không-dùng)
- [Trap 10: Why Design System?](#trap-10-why-design-system)
- [Trap 11: Why Abstraction?](#trap-11-why-abstraction)
- [Trap 12: Why NOT Abstraction?](#trap-12-why-not-abstraction)
- [Bảng tổng hợp Trap](#bảng-tổng-hợp-trap)

---

### Trap 1: Why React?

**Câu hỏi bẫy:** "Tại sao chọn React mà không phải Vue/Svelte?"

**Câu trả lời Junior sai:** "Vì React phổ biến, nhiều job, cộng đồng lớn, Facebook làm nên yên tâm."

> Sai vì: phổ biến không phải lý do kỹ thuật. Không nói được trade-off.

**Trả lời Senior đúng:**

- **Tại sao React:** 1) **Mental model** declarative + component + one-way data flow dễ scale, 2) **Hệ sinh thái** lớn (Next.js, RN, testing), 3) **Hiring** dễ, 4) **Concurrent Features** (Suspense, Transition) cho UX mượt, 5) **Stable** (backward compat tốt).
- **Trade-off:** React là **library, không phải framework** → phải tự chọn router, state, build. **Runtime overhead** (VDOM, reconciliation) nặng hơn Svelte (compile-time). **Abstraction leak** (closure stale, deps array).
- **Khi nào KHÔNG dùng:** Landing page tĩnh → Astro/HTML tốt hơn. App siêu nhẹ (<50kb) → Preact/Svelte. Team đã mạnh Vue và không cần RN → Vue cũng tốt, không cần đổi.

**Câu chốt Senior:** "Mình chọn React vì **team đã có kinh nghiệm, cần RN share logic, và hệ sinh thái Next.js cho SEO**. Nếu làm landing tĩnh thì mình sẽ chọn Astro."

---

### Trap 2: Why TypeScript?

**Câu hỏi bẫy:** "TypeScript có làm app chạy nhanh hơn không?"

**Câu trả lời Junior sai:** "Có, vì TypeScript tối ưu code nên chạy nhanh hơn JavaScript."

> Sai vì: TS chỉ là **compile-time**, runtime vẫn là JS, không nhanh hơn.

**Trả lời Senior đúng:**

- **Tại sao TS:** 1) **Catch bug sớm** (null, typo, sai contract) trước khi chạy, 2) **Refactor an toàn** (rename, move), 3) **DX** (autocomplete, jump to definition), 4) **Contract** giữa FE-BE (OpenAPI → type), 5) **Tài liệu sống** (type là doc).
- **Trade-off:** **Build chậm hơn** (type check), **config phức tạp** (`strict`, `paths`), **over-typing** (generic lồng nhau khó đọc), **false sense of safety** (vẫn cần runtime validation vì `any`, `as`, API drift).
- **Khi nào KHÔNG dùng:** Prototype 2 ngày, script nhỏ, team chưa quen → JS + JSDoc đủ. Dự án có quá nhiều `any` thì TS vô nghĩa.

**Code:**

```typescript
// TS không bảo vệ runtime
const data = JSON.parse(await fetch('/api/user').then(r=>r.text())) as User; // as che lỗi
// ✅ Cần zod
const user = UserSchema.parse(data);
```

---

### Trap 3: Why Next.js?

**Câu hỏi bẫy:** "Next.js có phải lúc nào cũng tốt hơn CRA/Vite?"

**Câu trả lời Junior sai:** "Có, vì Next.js có SSR nên SEO tốt hơn, nên luôn dùng Next.js."

> Sai vì: không phải app nào cũng cần SSR, SSR có cost.

**Trả lời Senior đúng:**

- **Tại sao Next.js:** 1) **Hybrid rendering** (SSR/SSG/ISR/CSR) linh hoạt, 2) **File-based routing + layouts**, 3) **Image/Font optimization** built-in, 4) **RSC** giảm JS, 5) **Deploy** dễ (Vercel), 6) **SEO** cho public page.
- **Trade-off:** **Complexity** (cache, revalidate, server/client component), **Lock-in** (deploy ngoài Vercel phức tạp hơn), **Build time** lâu với 100k page, **Debug khó** (server vs client).
- **Khi nào KHÔNG dùng:** Dashboard nội bộ (không cần SEO) → Vite SPA đủ, nhanh hơn. App cần **highly dynamic** (mọi thứ CSR) thì Next.js overhead thừa. Nếu team chỉ làm SPA và đã có infra Vite tốt thì không cần migrate.

**Decision:**

| Nhu cầu | Chọn |
|---|---|
| E-commerce, blog, marketing (SEO) | Next.js |
| Admin, dashboard nội bộ | Vite SPA |
| Static site ít JS | Astro |

---

### Trap 4: Why Redux / Zustand / Jotai?

**Câu hỏi bẫy:** "Redux đã lỗi thời chưa? Có nên bỏ Redux?"

**Câu trả lời Junior sai:** "Redux chết rồi, giờ ai cũng dùng Zustand, Redux boilerplate nhiều nên bỏ."

> Sai vì: Redux Toolkit đã gọn, và Zustand không thay thế được mọi use case Redux.

**Trả lời Senior đúng:**

- **Redux (RTK):** Tốt khi **state global phức tạp**, cần **time-travel, middleware, devtools mạnh**, team lớn cần **convention chặt**. Trade-off: boilerplate hơn (dù RTK đã giảm), bundle to hơn.
- **Zustand:** Tốt cho **global UI state** đơn giản (theme, cart, modal), **ít boilerplate**, **nhẹ**. Trade-off: không có built-in middleware/devtools mạnh như Redux.
- **Jotai/Recoil:** Tốt cho **atomic state** (mỗi atom độc lập, tránh re-render). Trade-off: mental model khác, ít người quen.
- **TanStack Query:** **Server state** (fetch, cache, sync) — không phải global UI state. Đừng dùng Redux để cache API.
- **Khi nào KHÔNG cần global state:** Dùng **URL** (filter), **local state** (`useState`), **Context** (theme), **Query** (server) là đủ. Chỉ 10% state cần global.

**Rule Senior:**

```
Server state → TanStack Query
URL state → useSearchParams
Local UI → useState
Global UI ít → Zustand
Global phức tạp + team lớn → Redux Toolkit
Atomic → Jotai
```

---

### Trap 5: Why Micro-frontend?

**Câu hỏi bẫy:** "Micro-frontend có phải là tương lai, nên áp dụng luôn?"

**Câu trả lời Junior sai:** "Có, vì micro-frontend như microservice, tách ra deploy độc lập nên luôn tốt."

> Sai vì: micro-frontend có **cost rất cao**, chỉ hợp khi **scale tổ chức**, không phải scale kỹ thuật.

**Trả lời Senior đúng:**

- **Tại sao:** 1) **Team độc lập** deploy (5 team x 50 dev), 2) **Tech stack khác nhau** (team A React, team B Vue), 3) **Release isolation** (team A deploy không block team B).
- **Trade-off:** **Bundle duplicate** (React 2 lần), **UX không nhất quán**, **Performance** kém (nhiều entry), **Complexity** (routing, auth, share state, deploy), **Debugging khó**.
- **Khi nào KHÔNG dùng:** Team < 20 dev, 1 product, 1 stack → **Monolith + Module Federation** hoặc **monorepo + feature-sliced** đủ. Đừng vì "muốn thử" mà trả giá.

**Alternative:** Monorepo + `pnpm workspace` + `Module Federation` cho share, hoặc đơn giản là **feature-based monolith**.

---

### Trap 6: Why Monorepo?

**Câu hỏi bẫy:** "Monorepo có tốt hơn polyrepo không?"

**Câu trả lời Junior sai:** "Monorepo tốt hơn vì mọi thứ trong 1 repo, dễ quản lý."

> Sai vì: monorepo cũng có trade-off, không phải lúc nào cũng tốt.

**Trả lời Senior đúng:**

- **Tại sao monorepo:** 1) **Share code** (UI kit, utils, types) dễ, 1 PR sửa cả FE/BE, 2) **Atomic change**, 3) **Tooling chung** (lint, CI), 4) **Refactor cross-package** dễ.
- **Trade-off:** **CI chậm** (phải affect graph), **Git to** (clone lâu), **Ownership** mờ (ai sở hữu file nào?), **Build cache** phức tạp (cần Turborepo/Nx).
- **Khi nào KHÔNG dùng:** 2 repo độc lập, ít share, team khác công ty → polyrepo đơn giản hơn. Monorepo cần **tooling** (Turborepo, Nx, pnpm).

**Stack:** `pnpm + Turborepo + changesets` là chuẩn 2024-2026.

---

### Trap 7: Why SSR / SSG / ISR?

**Câu hỏi bẫy:** "SSR có làm app nhanh hơn CSR không?"

**Câu trả lời Junior sai:** "Có, SSR luôn nhanh hơn vì render ở server."

> Sai vì: SSR **TTFB chậm hơn** (server render), chỉ **FCP/LCP nhanh hơn** nếu làm đúng, và có **hydration cost**.

**Trả lời Senior đúng:**

- **SSR:** HTML sẵn → **LCP nhanh, SEO tốt**, nhưng **TTFB chậm** (server phải render), **server cost**, **hydration JS vẫn phải tải**. Dùng cho **page cần SEO + dynamic** (product detail).
- **SSG:** Build-time HTML → **nhanh nhất, CDN cache**, nhưng **stale** đến khi rebuild. Dùng cho **blog, docs, marketing**.
- **ISR:** SSG + revalidate → **vừa nhanh vừa tươi**, nhưng **stale window** (60s). Dùng cho **e-commerce listing**.
- **CSR:** Chỉ JS → **TTFB nhanh** (HTML rỗng), nhưng **LCP chậm**, **SEO kém**. Dùng cho **dashboard**.
- **Khi nào KHÔNG SSR:** Dashboard, app sau login → CSR + Query đủ, SSR chỉ thêm phức tạp.

**Chọn:**

```
SEO + dynamic + ít đổi → ISR
SEO + static → SSG
SEO + realtime → SSR
No SEO + app → CSR
```

---

### Trap 8: Why Tailwind?

**Câu hỏi bẫy:** "Tailwind có phải best CSS framework?"

**Câu trả lời Junior sai:** "Có, vì Tailwind nhanh, utility-first, không cần viết CSS."

> Sai vì: Tailwind cũng có trade-off, không phải lúc nào cũng best.

**Trả lời Senior đúng:**

- **Tại sao Tailwind:** 1) **Velocity** (không đặt tên class, không switch file), 2) **Consistency** (design token sẵn), 3) **Bundle nhỏ** (purge), 4) **Responsive** nhanh (`md:`, `lg:`), 5) **Không lo specificity**.
- **Trade-off:** **HTML dài** (class 200 ký tự), **học lại** (nhớ class), **khó override** khi cần CSS phức tạp (animation, `:has()`), **lock-in** (muốn bỏ Tailwind phải rewrite).
- **Khi nào dùng:** Team mới, cần nhanh, design system chưa mature, app Tailwind-first.

> Update 2025: Tailwind v4 engine mới (Oxide), không cần tailwind.config.js, `@import "tailwindcss"`, Vite plugin `@tailwindcss/vite`. Trade-off cũ vẫn đúng.

---

### Trap 9: Why NOT Tailwind? (Khi nào không dùng)

**Câu hỏi bẫy:** "Khi nào bạn sẽ KHÔNG chọn Tailwind?"

**Câu trả lời Junior sai:** "Lúc nào cũng dùng Tailwind, không có lý do không dùng."

**Trả lời Senior đúng:**

- **Không dùng khi:** 1) **Design system đã có** với CSS Modules/Styled Components, 2) **Cần CSS phức tạp** (animation, grid, `::before` phức tạp) → CSS thuần dễ hơn, 3) **Team đã quen SCSS/BEM** và không muốn học lại, 4) **Cần theme runtime** đổi token động (Tailwind build-time), 5) **Email template** (Tailwind không support).
- **Alternative:** `CSS Modules + PostCSS`, `Vanilla Extract`, `Panda CSS` (type-safe), `UnoCSS`.

**Quyết định:** Nếu **design system = component library** (shadcn/ui) thì Tailwind hợp. Nếu **design system = token + CSS variable** thì CSS Modules + variable linh hoạt hơn.

---

### Trap 10: Why Design System?

**Câu hỏi bẫy:** "Có cần design system cho app 10 trang không?"

**Câu trả lời Junior sai:** "Có, vì design system giúp UI đẹp và chuyên nghiệp."

**Trả lời Senior đúng:**

- **Tại sao:** 1) **Consistency** (1 nút 1 style), 2) **Velocity** (không làm lại nút 10 lần), 3) **Maintainability** (đổi màu 1 chỗ), 4) **Onboarding** (dev mới biết dùng gì).
- **Trade-off:** **Cost cao** (xây, doc, maintain), **over-engineering** cho app nhỏ, **governance** (ai quyết?).
- **Khi nào KHÔNG cần:** App 10 trang, 2 dev, deadline 1 tháng → **copy-paste + Tailwind** nhanh hơn. Chỉ cần **3-5 component chung** (Button, Input, Modal) chứ không cần full system.
- **Khi nào cần:** > 3 team, > 50 trang, cần **brand consistency**, hoặc **multi-product**.

**Bắt đầu nhỏ:** `Button`, `Input`, `Dialog` → `Storybook` → `Figma token` → scale dần.

---

### Trap 11: Why Abstraction?

**Câu hỏi bẫy:** "Abstraction có phải luôn tốt? Code càng abstract càng senior?"

**Câu trả lời Junior sai:** "Có, abstract giúp tái sử dụng, DRY, nên abstract càng nhiều càng tốt."

**Trả lời Senior đúng:**

- **Tại sao abstraction:** 1) **DRY** (không lặp), 2) **Encapsulation** (che phức tạp), 3) **Testability** (mock dễ).
- **Trade-off:** **Indirection** (đọc code phải nhảy 3 file), **Premature abstraction** (abstract khi chỉ có 1 use case → sai), **Cost** (thêm layer).
- **Rule Senior:** **Rule of Three** — chỉ abstract khi **thấy lặp 3 lần** với pattern rõ. Lần 1 copy, lần 2 copy, lần 3 mới abstract.

```typescript
// ❌ Premature: abstract khi chỉ 1 use case
function useFetch<T>(url: string) { /* 50 dòng generic */ }
// Dùng 1 chỗ → overhead

// ✅ Đủ 3 chỗ mới abstract
// Khi thấy 3 component đều fetch + loading + error giống nhau → mới tách useFetch
```

---

### Trap 12: Why NOT Abstraction?

**Câu hỏi bẫy:** "Khi nào bạn sẽ chọn duplicate thay vì abstraction?"

**Câu trả lời Junior sai:** "Không bao giờ duplicate, DRY là nguyên tắc."

**Trả lời Senior đúng:**

- **Chọn duplicate khi:** 1) **Chưa rõ pattern** (2 chỗ trông giống nhưng sẽ diverge), 2) **Abstraction làm code khó hiểu hơn** (phải đọc 3 file để hiểu 1 flow), 3) **Coupling** (đổi 1 chỗ vỡ chỗ kia), 4) **YAGNI** (You Aren't Gonna Need It).
- **Triết lý Senior:** **"Duplication is cheaper than wrong abstraction"** (Sandi Metz). Thà duplicate 2 lần còn hơn abstract sai rồi phải gỡ.

**Ví dụ:**

```typescript
// 2 form trông giống nhưng business khác → đừng gộp
// OrderForm và UserForm có 2 field giống nhưng validate khác → tách, không abstract thành GenericForm
```

**Dấu hiệu over-abstraction:** Prop `mode: 'a' | 'b' | 'c'` với `if (mode === 'a')` khắp nơi → nên tách 2 component.

---

## Bảng tổng hợp Trap

| Trap | Junior sai | Senior đúng | Khi nào KHÔNG dùng |
|---|---|---|---|
| **Why React?** | Vì hot | Declarative + ecosystem + RN, nhưng runtime cost | Landing tĩnh → Astro |
| **Why TS?** | Chạy nhanh hơn | Catch bug + DX, nhưng build chậm, cần runtime validation | Prototype 2 ngày |
| **Why Next.js?** | Luôn tốt hơn | Hybrid rendering + SEO, nhưng complexity + lock-in | Dashboard nội bộ → Vite |
| **Why Redux?** | Redux chết | RTK cho complex, Zustand cho simple, Query cho server | Local/URL đủ thì không global |
| **Why Micro-frontend?** | Tương lai | Chỉ khi 5 team 50 dev, còn lại monolith | Team <20 → monolith |
| **Why Monorepo?** | Luôn tốt | Share + atomic, nhưng CI chậm, cần Turborepo | Ít share → polyrepo |
| **Why SSR?** | Luôn nhanh hơn | LCP nhanh nhưng TTFB chậm + hydration cost | Dashboard → CSR |
| **Why Tailwind?** | Best | Velocity + consistency, nhưng HTML dài | Design system có sẵn → CSS Modules |
| **Why NOT Tailwind?** | Luôn dùng | Khi cần CSS phức tạp, theme runtime | Email, animation phức tạp |
| **Why Design System?** | Cho đẹp | Consistency + velocity, nhưng cost cao | App 10 trang → 3 component đủ |
| **Why Abstraction?** | Càng nhiều càng tốt | DRY khi lặp 3 lần, còn lại overhead | Khi chỉ 1-2 use case |
| **Why NOT Abstraction?** | Không duplicate | Duplicate rẻ hơn sai abstraction | Khi pattern chưa rõ |

> **Tư duy Senior cho mọi trap:** Không có silver bullet. Mỗi công nghệ là **trade-off**. Câu trả lời hay luôn có 3 phần: **Tại sao dùng → Giá phải trả → Khi nào KHÔNG dùng**.
